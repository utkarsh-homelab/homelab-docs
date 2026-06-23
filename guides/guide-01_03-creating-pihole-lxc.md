# 01-03: Setting Up Pi-hole v6 + Unbound on Proxmox LXC - Network-Wide DNS

## Prerequisites

- [ ] Proxmox installed and web UI accessible
- Router admin access (to change DHCP DNS settings)

## What is Pi-hole?

Every device on your network makes hundreds of DNS queries a day. By default, those queries go to your ISP or a third-party provider like Google or Cloudflare - and they can see every domain you visit.

Pi-hole intercepts those queries at the network level, blocking ads, trackers, and telemetry domains before they ever leave your network. Pair it with Unbound, a recursive DNS resolver, and your queries go directly to the authoritative DNS servers — no third party in the middle.

We'll also set up local DNS records. Every service we deploy will get a friendly name like pve.lab.uttutu instead of an IP address.

## Why LXC Instead of Docker or VM?

| Method | Advantages | Disadvantages |
|--------|-----------|---------------|
| **LXC** (our choice) | Near-native performance, lowest overhead, Proxmox integrated | Linux-only |
| **Docker** | Easy to deploy, well-documented | Another container runtime, slightly more overhead |
| **VM** | Full isolation | ~300MB RAM overhead, slower to boot |

> LXC vs Docker
> - **LXC**: System container — runs a full OS (init, systemd, networking). Like a lightweight VM.
> - **Docker**: Application container — runs a single process. Not designed for persistent system services.
> 
> Pi-hole runs best as an LXC because it needs systemd, persistent DNS resolution, and tight network integration.

## Step 1: Download the LXC Template

In Proxmox web UI:

1. Go to **pve → local (storage) → CT Templates**
2. Click **Templates**
3. Search for `ubuntu-26`
4. Select **ubuntu-26.04-standard**
5. Click **Download**

Or via CLI:

```bash
pveam update
pveam available | grep ubuntu-26
pveam download local ubuntu-26.04-standard_26.04-1_amd64.tar.zst
```

## Step 2: Create the Pi-hole Container

In Proxmox web UI:
1. Click **Create CT** (top right)

### General

| Field | Value |
|-------|-------|
| Node | **pve** |
| CT ID | **101** |
| Hostname | **pihole** |
| Password | (choose a password) |
| Unprivileged container | **✔ (checked)** |

> [!info] Unprivileged Container
> Unprivileged containers map the container's root (UID 0) to an unprivileged host UID (100000). This means a container breakout still gets a non-root user on the host. **Always use unprivileged containers.**

### Template

| Field | Value |
|-------|-------|
| Storage | **local** |
| Template | **ubuntu-26.04-standard_26.04-1_amd64.tar.zst** |

### Disks

| Field | Value | Why |
|-------|-------|-----|
| Storage | **local-lvm** |
| Disk size | **4 GB** | Pi-hole is tiny — 4GB is generous |

### CPU

| Field | Value |
|-------|-------|
| Cores | **1** |

### Memory

| Field | Value | Why |
|-------|-------|-----|
| Memory | **512 MB** | Pi-hole uses ~200MB idle |
| Swap | **512 MB** | |

### Network

| Field | Value |
|-------|-------|
| Name | **eth0** |
| Bridge | **vmbr0** |
| IPv4 | **DHCP** gets 192.168.0.101 (or your chosen static IP) |
| IPv6 | **Disabled** |

> [!tip] Give Pi-hole a Static IP
> DNS must have a fixed IP so devices know where to find it. After creation, we'll make it static:
> - `192.168.0.101/24`
> - Gateway: `192.168.0.1`
> - DNS: `1.1.1.1` (for reaching the internet before Pi-hole itself is running)

### Confirm

- Review settings
- Click **Finish**

## Step 3: Start and Configure the Container

```bash
# Start the container
pct start 101

# Enter the container
pct enter 101

# Or from Proxmox web UI: select CT 101 → Console
```

### Update the Container

```bash
apt update && apt upgrade -y
apt install curl wget git -y
```

## Step 4: Install Pi-hole

Pi-hole's installer is interactive — it uses dialog prompts to walk you through each configuration choice. These dialogs require a proper terminal that pct enter via SSH doesn't always provide. Use the PVE web console for the install.

```bash
# Pi-hole automated install
curl -sSL https://install.pi-hole.net | bash
```

### Installer Prompts

| Prompt | Answer | Why |
|--------|--------|-----|
| Select interface | **eth0** | Our container's network interface |
| Upstream DNS | **Cloudflare (1.1.1.1)** | Fast, privacy-focused |
| Block lists | **Default** (Stevenson's list) | Blocks most ads |
| IPv4 | **Yes** | We use IPv4 |
| IPv6 | **No** | Not needed for homelab |
| Static IP confirmation | **Yes** | Keep `192.168.0.101` |
| Web admin interface | **Yes** | Port 80 admin UI |
| Logging | **Show queries** | Useful for debugging |

### Post-Install

Installation completes with:
- **Web Interface**: `http://192.168.0.101/admin`
- **Password**: (randomly generated — you can change it by running `pihole setpassword`)

### Verify Pi-hole Running

```bash
# Check DNS resolution
dig google.com @127.0.0.1
▶ ;; Query time: 25 msec
  ;; ANSWER SECTION:
  google.com.     300     IN      A       142.250.67.78

# Check Pi-hole status
pihole status
▶ [✓] DNS service is running
```

## Step 5: Explore the Web UI

Open a browser and navigate to http://<your-pihole-ip>/admin. Log in with the password from the installer.

The first thing you'll notice is that Domains on Lists shows Error (-2). This is normal — the gravity database hasn't been populated yet.

Go to Tools > Update Gravity and click Update. This downloads the blocklists and populates the database. Without this, Pi-hole is running but not actually blocking anything.

Pi-hole v6's web interface:

- Dashboard — query stats, top blocked domains, query types, and upstream destinations
- Query Log — real-time log of every DNS query hitting Pi-hole
- Domains — manage allow and blocklists
- Local DNS — where you'll add records for services like pbs.hake.rodeo
- Settings — DNS configuration, privacy, API access, and more

## Step 6: Configure Your Network to use Pi-Hole

Now that Pi-hole is running with Unbound, update your containers and VMs to use it for DNS.

### Update Containers

For LXC containers, pct set updates the DNS directly — no reboot needed:

```bash
pct set 101 --nameserver 127.0.0.1
```

Pi-hole's own container uses 127.0.0.1 since it queries itself. For other containers, use Pi-hole's IP:

```bash
pct set 102 --nameserver 192.168.0.101
```

Repeat for each container you want to use Pi-hole. Replace 102 with your container IDs.

### Update VMs

VMs are different from containers — PVE doesn't control the VM's DNS directly. For VMs that were installed from their own ISO, you need to update DNS inside the VM.

SSH into the VM and edit /etc/resolv.conf:

```bash
nano /etc/resolv.conf
```

Change the nameserver line to Pi-hole's IP, then save

```
nameserver 192.168.0.101
```

This persists across reboots unless something like dhcpcd overwrites it.

### Update Your PC

Depends on your system for me it's the following, I'm using Arch btw :)

#### Changing DNS using the Omarchy Menu

- Press `SUPER + ALT + SPACE` to open the Omarchy launcher menu.
- Navigate to `Setup > DNS`.
- Choose the Custom option and enter PiHole IP (for me it's 192.168.0.101)

### Permanently change DNS server for all devices (via DHCP on the Router):

You can configure your ISP router to hand out Pi-hole as DNS:

1. Log into ISP router (for me it's `http://192.168.0.1`)
2. Find DHCP settings
3. Set DNS Server to PiHole IP (for me it's `192.168.0.101`)
4. Save and reload DHCP

## Step 7: Test Ad Blocking

```bash
# Test a blocked domain
dig doubleclick.net @192.168.0.101
▶ ;; ANSWER: 0.0.0.0  ← Blocked!

# Test an allowed domain
dig google.com @192.168.0.101
▶ ;; ANSWER: 142.250.67.78  ← Allowed!

# Check Pi-hole dashboard: http://192.168.0.101/admin
```

## Step 8: Configure Pi-hole Updates

```bash
# Pi-hole can self-update:
pihole updatePihole

# Or add to cron for weekly updates:
crontab -e
# Add:
@weekly /usr/local/bin/pihole updatePihole
```

## Step 9: Install and Configure Unbound

### What is Unbound

Unbound is a recursive DNS resolver. Instead of forwarding your queries to Cloudflare or Google, it resolves them by going directly to the authoritative DNS servers — starting from the root servers and working down the chain. No third party ever sees your full query.

### Update the container and install Unbound:
```bash
sudo apt update
sudo apt install -y unbound dnsutils
```

### Create the Unbound config file:

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

Use this config:

```text
server:
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    prefer-ip6: no
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 1
    so-rcvbuf: 1m
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

The key settings:

- port: 5335 — avoids the port 53 conflict with Pi-hole
- harden-dnssec-stripped: yes — enables DNSSEC validation, rejecting responses with invalid signatures
- prefetch: yes — refreshes popular domains before they expire from cache, keeping queries fast
- private-address blocks — prevents DNS rebinding attacks by refusing to return private IP addresses from public queries

### Then restart and enable Unbound:

```bash
sudo systemctl restart unbound
sudo systemctl enable unbound
```

This local-port pattern is the standard Pi-hole + Unbound setup, and Pi-hole’s own documentation uses port 5335 for Unbound.

### Connect Pi-hole :

In the Pi-hole admin UI, go to Settings → DNS and set the custom upstream to:

```text
127.0.0.1#5335
```

Disable the other upstream resolvers if you want all queries to go through Unbound only. 
That keeps Pi-hole forwarding locally to Unbound instead of using the router or public DNS servers.
The `#5335` tells Pi-hole to use Unbound's port instead of the default `53`.

### Interface Settings :

Scroll down to Interface settings. This controls which devices Pi-hole responds to.

The options are:

- Allow only local requests — responds to queries from devices on the same subnet only (one hop away)
- Respond only on interface eth0 — binds to eth0 specifically
- Bind only to interface eth0 — stricter binding
- Permit all origins — responds to queries from any source

If all your devices and containers are on the same subnet as Pi-hole, Allow only local requests is the right choice — it's the most restrictive option that works.

If you're querying Pi-hole from a different VLAN or subnet (e.g., your PC is on VLAN 10 but Pi-hole is on VLAN 20), you'll need Permit all origins. Since Pi-hole is internal-only with no internet exposure, there's no security risk either way.

## Step 10: Local DNS Records

Local DNS records let you give services friendly names instead of IP addresses. Instead of accessing PVE at https://192.168.0.100:8006, we can use pve.lab.uttutu:8006.

In the Pi-hole web UI, go to Local DNS > DNS Records.

### Add a record:

Domain: pve.lab.uttutu
IP Address: 192.168.0.100
Click Add. The record appears immediately.

### Verify it resolves:

```
dig pve.lab.uttutu @192.168.0.101
```

You should see 192.168.0.100 in the answer section. 
We can add more records for any other services you want to access by name. (I'll probably automate this in the future)

## References

- [Pi-hole Official Documentation](https://docs.pi-hole.net/)
- [Pi-hole Blocklists](https://firebog.net/)
- [Proxmox LXC Container Guide](https://pve.proxmox.com/wiki/Linux_Container)