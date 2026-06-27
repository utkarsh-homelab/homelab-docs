# 01-06: Proxmox NFS Share + Firewall Setup

## Prerequisites

- [ ] Proxmox installed and running
- [ ] SSH access to Proxmox host as root
- [ ] Kubernetes cluster running

> Kubernetes cluster setup
>
> refer to [guide-02_01](./guide-02_01-creating-kubernetes-cluster.md) for manual setup.
> OR
> refer to [guide-02_02](./guide-02_02-automating-k8s-vm-creation-terraform.md) and [guide-02_03](./guide-02_03-automating-k8s-cluster-setup-ansible.md) for automated setup.

## What We're Building

An NFS share on the Proxmox host's root filesystem that our Kubernetes cluster will use for persistent volume storage via `csi-driver-nfs`. 

> I won't be automating this setup since I'm planning to move the persistent storage to a NAS that I'll be building in future, so this will work as our storage only for now.

We'll also lock down the Proxmox host with iptables to allow only essential traffic.

## Step 1: Install NFS Server on Proxmox

```bash
# SSH into Proxmox host
ssh root@<proxmox-ip>

# Install NFS server and iptables-persistent
apt update
apt install -y nfs-kernel-server iptables-persistent
```

## Step 2: Create the NFS Export Directory

```bash
# Create the export directory
mkdir -p /var/nfs/kubernetes

# Set ownership to nobody:nogroup (NFS convention for anonymous access)
chown -R nobody:nogroup /var/nfs/kubernetes
chmod 755 /var/nfs/kubernetes
```

## Step 3: Configure the NFS Export

Edit `/etc/exports`:

```bash
# Use your subnet range
echo "/var/nfs/kubernetes 192.168.0.0/24(rw,async,no_subtree_check,no_root_squash)" >> /etc/exports
```

| Option | Why |
|--------|-----|
| `rw` | Read-write access for Kubernetes pods |
| `async` | Better performance (safe with UPS battery backup) |
| `no_subtree_check` | Reduces NFS operation overhead on the server |
| `no_root_squash` | Allows pods running as root to write files as root (required for some StatefulSets) |

### Apply and Verify

```bash
# Use your subnet
# Apply the export configuration
exportfs -arv
▶ exporting 192.168.0.0/24:/var/nfs/kubernetes

# Enable and start NFS server
systemctl enable --now nfs-server

# Verify the share is available
showmount -e localhost
▶ Export list for localhost:
▶ /var/nfs/kubernetes 192.168.0.0/24 
```

### Verify NFS Works from a Kubernetes Node

```bash
# SSH into k8s-master-1
ssh ubuntu@<k8s-master-1-ip>

# Install nfs-common (needed for mounting)
sudo apt install -y nfs-common

# Test mount
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs <proxmox-ip>:/var/nfs/kubernetes /mnt/test-nfs

# Test file creation and deletion
echo "Hello from NFS" | sudo tee /mnt/test-nfs/hello.txt
ls -la /mnt/test-nfs/
sudo cat /mnt/test-nfs/hello.txt
sudo rm -rf /mnt/test-nfs/hello.txt

# Unmount
sudo umount /mnt/test-nfs
sudo rm -rf /mnt/test-nfs
```

If you can create and read back the file, the NFS share works.

## Step 4: Set Up iptables Firewall

We'll configure iptables to allow only essential services and restrict NFS access to our local subnet. 

> replace the subnet values with your subnet values.

### Create the Firewall Rules

```bash
# Flush existing rules (start clean)
iptables -F INPUT
iptables -F FORWARD
iptables -F OUTPUT

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow established/related connections (so return traffic works)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow loopback (localhost)
iptables -A INPUT -i lo -j ACCEPT

# Allow SSH (port 22) — from entire local network
iptables -A INPUT -s 192.168.0.0/24 -p tcp --dport 22 -j ACCEPT

# Allow Proxmox Web UI (port 8006)
iptables -A INPUT -s 192.168.0.0/24 -p tcp --dport 8006 -j ACCEPT

# Allow Proxmox API/Cluster traffic (for clustering if you add more nodes later)
iptables -A INPUT -s 192.168.0.0/24 -p tcp --dport 111 -j ACCEPT  # rpcbind
iptables -A INPUT -s 192.168.0.0/24 -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s 192.168.0.0/24 -p tcp --dport 2049 -j ACCEPT # NFS
iptables -A INPUT -s 192.168.0.0/24 -p udp --dport 2049 -j ACCEPT
iptables -A INPUT -s 192.168.0.0/24 -p tcp --dport 20048 -j ACCEPT # mountd
iptables -A INPUT -s 192.168.0.0/24 -p udp --dport 20048 -j ACCEPT

# Allow ICMP (ping) for network debugging
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Allow DNS outbound (needed for resolving package repos)
iptables -A INPUT -p udp --sport 53 -j ACCEPT
iptables -A INPUT -p tcp --sport 53 -j ACCEPT

# Allow HTTP/HTTPS for apt updates (response traffic)
iptables -A INPUT -p tcp --sport 80 -m conntrack --ctstate ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp --sport 443 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

### Save the Rules Persistently

```bash
# Save rules so they survive reboot
iptables-save > /etc/iptables/rules.v4

# Verify the saved rules
iptables -L -n -v
```

### Change nfs configuration to only use 20048 for mountd

The NFS export will work now, but `showmount -e <proxmox-ip>` will not return the export list because `rpc.mountd` is using dynamic high ports instead of a fixed port. 
The firewall allows TCP/UDP 111 and 2049, plus 20048, but `mountd` is not actually listening on 20048, so RPC export discovery will fail.
Even when `showmount -e` fails (because `showmount` depends on the MOUNT RPC service), the NFS data path will still be reachable and functional. 
But we need `showmount` to also work for quick testing if the connection works without actually mounting anything if need arises.

On Proxmox/Debian, the effective setting came from `/etc/nfs.conf` under the `[mountd]` section, where `port=0` meant dynamic port assignment. Setting a fixed port will make `mountd` predictable and firewall-friendly.

```text
[mountd]
manage-gids=y
port=20048

```

After updating the config, restart NFS services and re-check the registered RPC ports 


```bash
systemctl restart rpcbind nfs-server
exportfs -ra
rpcinfo -p | grep mountd
```

Expected result: mountd should show port 20048 instead of random high ports 

### Test the Firewall

```bash
# From your laptop — SSH should work
ssh root@<proxmox-ip>

# From your laptop — Web UI should work
# Open browser to https://<proxmox-ip>:8006

# From outside the subnet (simulate by using a non-matching IP) —
# This should timeout/be rejected
# But since we're on the same subnet, everything should be fine

# Verify NFS is still accessible from k8s-master-1
ssh ubuntu@<k8s-master-1-ip>
showmount -e <proxmox-ip>
▶ Export list for <proxmox-ip>:
▶ /var/nfs/kubernetes 192.168.0.0/24 # your subnet
```

## Step 5: Verify NFS Ports Are Listening

```bash
# On Proxmox host
ss -tlnp | grep -E '(2049|111|20048)'
▶ LISTEN   0   64    0.0.0.0:2049   0.0.0.0:*   users:(("nfsd",pid=...))
▶ LISTEN   0   64    0.0.0.0:111    0.0.0.0:*   users:(("rpcbind",pid=...))
```

## (Optional) Step 6: Connecting to the nfs-share from your machine

> I wanted to connect to the nfs share from my laptop so I'm adding the steps to do so here, you don't have to do this if you don't want.
>
> I'm using arch btw ;)
> So the steps here are for Arch linux, if you are using a different linux distro subtitute the package manager accordingly.

### Install NFS Utilities

```bash
sudo pacman -S nfs-utils
```

### Discover Remote Shares (Optional)

```bash
showmount -e <proxmox-ip>
```

### Create a Local Mount Point

```bash
sudo mkdir -p /mnt/nfs_proxmox
```

### Mount the Share Manually

```bash
# sudo mount -t nfs <proxmox-ip>:/path/to/remote/share /mnt/nfs_share
sudo mount -t nfs 192.168.0.100:/var/nfs/kubernetes /mnt/nfs_proxmox
```

### Configure Permanent Automounting (Recommended)

To prevent your system from hanging during boot if the network is not ready, we'll use `systemd` automount flags inside our `/etc/fstab` file

#### Open the file layout via a text editor:

```bash
sudo nano /etc/fstab
```

#### Append the following configuration line to the bottom of the file:

Example :
```
<SERVER_IP>:/path/to/remote/share /mnt/nfs_share nfs defaults,noauto,x-systemd.automount,x-systemd.device-timeout=10,timeo=14,x-systemd.idle-timeout=1min 0 0
```

For me it's:
```
192.168.0.100:/var/nfs/kubernetes /mnt/nfs_proxmox nfs defaults,noauto,x-systemd.automount,x-systemd.device-timeout=10,timeo=14,x-systemd.idle-timeout=1min 0 0
```

#### Save the changes and reload systemd configuration

```bash
sudo systemctl daemon-reload
sudo systemctl restart remote-fs.target
```


## Summary

| Component | Value |
|-----------|-------|
| NFS Server | 192.168.0.100 (Proxmox host) |
| NFS Export | `/var/nfs/kubernetes` |
| NFS Options | `rw,async,no_subtree_check,no_root_squash` |
| Allowed Subnet | 192.168.0.0/24 |
| Firewall | iptables with persistent rules |
| NFS Ports | 111 (rpcbind), 2049 (nfs), 20048 (mountd) |

## Next Steps

The NFS share is ready. Later, we'll deploy `csi-driver-nfs` on the kubernetes cluster to use this share for dynamic PVC provisioning.

## References

- [NFS on Debian/Ubuntu](https://ubuntu.com/server/docs/service-nfs)
- [iptables-persistent](https://help.ubuntu.com/community/IptablesHowTo)
- [CSI Driver NFS](https://github.com/kubernetes-csi/csi-driver-nfs)
