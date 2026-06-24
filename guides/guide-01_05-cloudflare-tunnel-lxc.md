# 01-05: Creating a Cloudflare Tunnel in LXC on Proxmox 

## Prerequisites

- Proxmox installed and web UI accessible
- Cloudflare account
- A Domain name

## Why?

We are deploying cloudflared in a lightweight LXC container to securely expose homelab services to the internet without opening any firewall ports.

## What is Cloudflare Tunnel?

Cloudflare Tunnel creates an **encrypted outbound tunnel** from your server to Cloudflare's edge network. Instead of opening a port in your firewall and hoping for the best, cloudflared connects out to Cloudflare, and Cloudflare routes traffic back through that connection.

> [!info] Zero Trust Security Model
> Traditional security assumes "inside the network is safe, outside is dangerous." Zero Trust assumes **no network is safe** — every request must be authenticated and authorized, regardless of origin.
>
> Cloudflare Tunnel implements Zero Trust by:
> 1. No public IP → attackers can't find your server
> 2. No open ports → attackers can't scan or probe
> 3. Identity verification → users must authenticate (Google OAuth, etc.) before reaching the tunnel
> 4. Device posture → optional checks (OS version, disk encryption, antivirus running)

## Step 1: Install cloudflared in LXC

### 2.1 Create the LXC Container

Use the UI or the command line, I'll use the command line

```bash
# Create container for Cloudflare Tunnel
pct create 102 \
  /var/lib/vz/template/cache/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname cftunnel \
  --storage local \
  --rootfs containers:2 \
  --memory 256 \
  --swap 256 \
  --cores 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.0.102/24,gw=192.168.0.1 \
  --nameserver 192.168.0.101 \
  --searchdomain lab.uttutu \
  --unprivileged 1 \
  --start 1
```

> [!info] Resource Allocation Notes
> - **256MB RAM**: cloudflared needs very little memory. Most of the time, it's idle with an open WebSocket connection.
> - **256MB swap**: Extra safety net for memory spikes during tunnel reconnection.
> - **2GB rootfs**: Minimal OS + cloudflared fits in under 1GB.
> - **DNS → 192.168.0.101**: The tunnel container uses Pi-hole for DNS. This ensures it can resolve `.lab.uttutu` hostnames (like `pve.lab.uttutu`) for routing.

### 2.2 Install cloudflared

```bash
pct enter 102

# Update and install dependencies
apt update && apt full-upgrade -y
apt install -y curl wget

# Download cloudflared (official package)
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o /tmp/cloudflared.deb
dpkg -i /tmp/cloudflared.deb

# Verify installation
cloudflared version
▶ cloudflared version 2026.6.1 (built 2026-06-18-14:33 UTC)

# Exit container
exit
```

## Step 2: Cloudflare Account Setup


### 1.1 Add Your Domain to Cloudflare

If you haven't already:
1. Go to [dash.cloudflare.com](https://dash.cloudflare.com)
2. **Add a Site** → enter your domain
3. Select the **Free plan**
4. Cloudflare scans your existing DNS records
5. Change your domain's nameservers at your registrar to Cloudflare's
6. Wait for propagation (5 minutes to 48 hours — usually under 1 hour)

> [!info] What Happens When You Move DNS to Cloudflare
> Cloudflare becomes your **authoritative DNS provider** and **reverse proxy**:
> 1. Visitors query Cloudflare DNS → get Cloudflare edge IPs
> 2. Cloudflare proxies requests (caching, DDoS filtering, WAF) → forwards to your origin
> 3. Your origin IP is hidden — attackers only see Cloudflare IPs
>
> This is called **proxied mode** (orange cloud in Cloudflare DNS). Without it, visitors connect directly to your server (gray cloud).

### 1.2 Create Tunnel on Cloudflare

- Select Zero Trust from the Cloudflare Dashboard
- Select the Free plan and checkout with your payment details 

> Cloudflare Zero Trust requires a valid credit card on file, even for the Free plan. Cloudflare uses this to prevent abuse, verify identity, and authorize the creation of a Zero Trust team. You will not be charged unless you exceed the free limits (such as adding more than 50 users)

- Go to Networks > Connectors > Add a tunnel > Select Cloudflared > Give a name to the tunnel I'll use `homelab-tunnel` > Click Save

It will show the steps to Install and run a connector choose your distro and architecture for me it's debian 64bit

We have already installed cloudflared in the LXC container so we only need to install a service to automatically run your tunnel whenever your machine starts

## Step 3: Run the Tunnel as a Service

Inside the LXC container run the command shown in the previous step

```bash
# Install as systemd service
sudo cloudflared service install <your-token>

# Verify the service file
cat /etc/systemd/system/cloudflared.service

# Start and enable
systemctl enable --now cloudflared

# Check status
systemctl status cloudflared
▶ cloudflared.service - cloudflared
▶    Loaded: loaded
▶    Active: active (running)
```

> [!tip] Systemd Service vs Manual Run
> Running cloudflared as a systemd service ensures:
> - Starts automatically on boot
> - Restarts automatically if it crashes
> - Logs to journald (`journalctl -u cloudflared -f`)
> - Doesn't depend on a user session

### Verify Tunnel Status

```bash
# Check tunnel status
cloudflared tunnel info homelab
▶ Status: active
▶ Connections: 1 (connecting to 2 cloudflare edges)
▶ Connector ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Live logs
journalctl -u cloudflared -f
▶ INF  Connection xxxxx registered  connIndex=0
▶ INF  Connection xxxxx registered  connIndex=1
```

## Step 4: Create a test route

Create a test route to test that the tunnel is forwarding traffic to your services 

I'll create a route with hostname `test.uttutu.xyz` that points to a test nginx web server that's running inside my kubernetes cluster and is exposed via a nodeport.

## Step 5: Configure Cloudflare Zero Trust (Access Policies)

The tunnel is running, but anyone who knows your URL can reach your services. Add authentication:

1. **Cloudflare Zero Trust Dashboard → Access controls → Applications**
2. **Add an Application → Self-hosted**
3. **Configure**:

```text
Application name: Homelab
Domain: *.yourdomain.com
Subdomain: * (wildcard — covers all services)
Session Duration: 24h
```

4. **Add a Policy:**

```text
Policy name: Homelab-policy
Action: Allow
Include: Everyone (start with this, lock down later)

Or more restrictive:
Include: Emails ending in -> @gmail.com 
Include: Email in -> list of trusted emails
```

6. **Save**

> [!info] Zero Trust Access Flow
> ```
> User → https://<subdomain>.yourdomain.com
>     ↓
> Cloudflare Edge checks Access policy
>     ↓
> User redirected to Google OAuth login
>     ↓
> User authenticated → JWT set in cookie
>     ↓
> Cloudflare checks: Is this user allowed?
>     ↓
> Yes → Forward to tunnel
> ```
>
> The user never sees your origin. Even if they somehow bypass Cloudflare, the tunnel only accepts connections from Cloudflare edge IPs.

## Step 6: Secure the Tunnel Container

### 6.1 Firewall Rules

```bash
# Inside the container — cloudflared only needs outbound HTTPS
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 22 -j ACCEPT  # SSH only
iptables -A INPUT -j DROP

iptables -A OUTPUT -d 192.168.1.241 -p tcp --dport 443 -j ACCEPT  # Traefik
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -j ACCEPT  # Allow outbound

apt install -y iptables-persistent
netfilter-persistent save
```

### 6.2 Automatic Updates

```bash
apt install -y unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
```

> [!info] Security Considerations
> The tunnel container is your **internet-facing entry point**. Keep it minimal — only cloudflared and SSH are installed. No web server, no database, no tools. If someone compromises this container, they can:
> 1. Access your internal services via the tunnel (if credentials are in the config)
> 2. Disconnect the tunnel (cutting external access)
>
> But they can NOT:
> 1. Access other containers (LXC isolation)
> 2. Access Proxmox (separate authentication)
> 3. Access Kubernetes (RBAC + network policies)

## Step 7: Maintenance

### Updating cloudflared

```bash
cloudflared update

# Or manually:
dpkg -i /tmp/cloudflared-linux-amd64.deb
systemctl restart cloudflared
```

### Adding a New Service

When we deploy a new service that we want accesible to the internet:

We can to go to the Cloudflare Zero Trust Dashboard and add the route to the Published applications routes in our Connector settings.

## References

- [Linux Container](https://pve.proxmox.com/wiki/Linux_Container)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
