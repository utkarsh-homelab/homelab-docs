# 01-02: Creating First Ubuntu VM in Proxmox

## Prerequisites

- [ ] Proxmox installed and web UI accessible

## Step 1: Download Ubuntu Server ISO

In the Proxmox web UI:

1. Navigate to **pve → local (storage) → ISO Images → Download from URL**
2. Paste the download link for the ISO, 
(You can find the ISOs on [Ubuntu Release Page](https://releases.ubuntu.com/)) 
(I'll be using the 26.04 LTS version here's the link for it `https://releases.ubuntu.com/resolute/ubuntu-26.04-live-server-amd64.iso`)  
3. Click **Download** (wait for completion)

**Alternative**: Download via CLI:

```bash
# On Proxmox shell
wget -O /var/lib/vz/template/iso/ubuntu-26.04-live-server-amd64.iso \
  https://releases.ubuntu.com/resolute/ubuntu-26.04-live-server-amd64.iso
```

## Step 2: Create the VM

In Proxmox web UI:

1. Click **Create VM** (top right)
2. Fill in:

### General

| Field | Value | Why |
|-------|-------|-----|
| Node | **pve** | Our only node |
| VM ID | **100** | First VM, easy to remember |
| Name | **ubuntu-server** | Descriptive name |

### OS

| Field | Value |
|-------|-------|
| Storage | **local** (where ISO is stored) |
| ISO image | **ubuntu-26.04-live-server-amd64.iso** |
| Type | **Linux** |
| Version | **7.x - 2.6 Kernel** |

### System

| Field | Value | Why |
|-------|-------|-----|
| Graphics card | **SPICE** | For console access later |
| SCSI Controller | **VirtIO SCSI** | Best performance for Linux |
| Qemu Agent | **✔ Enable** | Allows Proxmox to communicate with VM for graceful shutdown |

> [!info] Qemu Guest Agent
> The Qemu Guest Agent runs inside the VM and lets Proxmox:
> - Gracefully shut down the VM (instead of power-off)
> - Get accurate IP/OS info
> - Freeze filesystem during snapshots
> Always enable this. Install it inside the VM later: `apt install qemu-guest-agent`

### Disks

| Field | Value | Why |
|-------|-------|-----|
| Bus/Device | **VirtIO Block** | Best performance for Linux |
| Storage | **local** | Uses NVMe storage |
| Disk size | **50 GB** | Depending on what you will be using the VM for |
| Cache | **Write back** | Better performance (safe with UPS) |

> [!info] VirtIO vs SATA vs IDE
> - **VirtIO**: Paravirtualized — the VM knows it's a VM and uses special drivers. Fastest.
> - **SATA**: Emulated SATA controller. Slower (driver overhead). More compatible.
> - **IDE**: Legacy. Very slow. Use only for old OSes (Windows XP).
> 
> For Linux VMs, always use VirtIO SCSI or VirtIO Block.

### CPU

| Field | Value | Why |
|-------|-------|-----|
| Sockets | **1** | Single CPU package |
| Cores | **2** | Depending on the use case, my processor has 6 cores only :( |
| Type | **host** | Expose host CPU features (better performance) |

> [!info] CPU Type: host vs kvm64
> - **host**: Passes through all CPU features (VT-x, AES-NI, AVX2). Best performance. VM can't be live-migrated to different CPU types.
> - **kvm64**: Emulates a generic CPU. Slower but VM can move between different physical CPUs.
> - **host** is fine for homelab (same CPU across all ProDesks).

### Memory

| Field | Value | Why |
|-------|-------|-----|
| Memory | **2048 MB** | Depending on your use case |

### Network

| Field | Value | Why |
|-------|-------|-----|
| Bridge | **vmbr0** | Our Proxmox bridge |
| Model | **VirtIO (paravirtualized)** | Best performance |
| VLAN Tag | **(leave empty)** | No VLANs in Phase 1 |

### Confirm

- Review all settings
- Uncheck **Start after created** (we need to do one more config first)
- Click **Finish**

## Step 3: (Optional) Configure Cloud-Init for Automatic Setup

Cloud-init automates VM initialization (hostname, SSH keys, network). I won't be doing it for this VM.


## Step 4: Install Ubuntu in the VM

1. Select VM 100 (ubuntu-server) in the left panel
2. Click **Start** (top right)
3. Click **Console** (opens the VM display)
4. Wait for Ubuntu installer to load
5. Select **Try or Install Ubuntu Server**
6. Follow the installer:

### Ubuntu Install Steps

| Step | Selection |
|------|-----------|
| Language | **English** |
| Keyboard | **English (US)** |
| Ubuntu Server | **✔ Continue** |
| Network interface | **ens18** (the VirtIO NIC) |
| DHCP | **Enabled** (gets IP from ISP router) |
| Proxy | **(leave blank)** |
| Mirror | **http://archive.ubuntu.com/ubuntu** |
| Storage | **Use entire disk (50GB)** — default partition scheme |
| Profile | Your name, server name, for me it's: `ubuntu-server`, username, , for me it's: `uttutu` |
| SSH setup | **✔ Install OpenSSH server** + **✔ Import SSH key** (paste your public key) |
| Featured snaps | **(none)** |
| Install | **Continue** |

> [!tip] SSH Key Setup
> During Ubuntu install, you can paste your laptop's public SSH key (`~/.ssh/id_ed25519.pub` or `~/.ssh/id_rsa.pub`). This lets you SSH without a password.
> 
> ```bash
> # On your laptop, get your public key:
> cat ~/.ssh/id_*.pub
> ```

**Wait ~5 minutes** for installation to complete.

5. Click **Reboot Now**
6. The VM will reboot into Ubuntu
7. Login with the username/password you created

## Step 5: Post-Install VM Configuration

```bash
# SSH into the VM (from your laptop)
ssh <username>@<vm-ip-address>

# Update packages
sudo apt update && sudo apt upgrade -y

# Install Qemu Guest Agent (for Proxmox integration)
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent

# Install common tools
sudo apt install -y curl wget git vim htop net-tools

# Set timezone
sudo timedatectl set-timezone Asia/Kolkata

# Verify guest agent works (on Proxmox shell)
# → VM should show IP and QEMU agent status as "running"
```

## Step 6: Verify from Proxmox

Back in Proxmox web UI:

1. Click on VM 100 (docker-node)
2. Go to **Summary** tab
3. You should see:
   - **Status**: Running
   - **IP**: `vm-ip-address`
   - **QEMU agent**: Running

## Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| VM stuck at "Booting from Hard Disk" | ISO still mounted | Remove ISO from VM's CD/DVD drive |
| VM resets after Ubuntu "Reboot Now" | Need to remove install media | Detach ISO in Hardware tab |
| Can't SSH to VM | SSH server not installed | In Proxmox console: `sudo apt install openssh-server` |
| VM shows "Not running" after creation | You need to start it | Select VM → Start button |
| Poor VM performance | CPU type is kvm64 (default) | Change CPU type to "host" |
| QEMU agent shows "not running" in Proxmox | Agent not installed or running | `sudo systemctl enable --now qemu-guest-agent` |
| VM can't ping google.com | DNS not set or no gateway | Check `/etc/netplan/01-netcfg.yaml` |

## References

- [Ubuntu Server 26.04 LTS Download](https://releases.ubuntu.com/26.04/)
- [Proxmox VM Creation Guide](https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines)
- [Cloud-Init Documentation](https://cloudinit.readthedocs.io/)
- Video: [Create Ubuntu VM in Proxmox](https://www.youtube.com/results?search_query=create+ubuntu+vm+proxmox)

