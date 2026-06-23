# 01-01 : Proxmox Installation

## Phase 1 : Creating a Bootable Proxmox USB

### Prerequisites

- A computer with internet access (my laptop)
- A USB drive (8GB minimum, 16GB recommended) (I have a 64GB amazon basics one )
- About 2GB of free disk space for the ISO download

### Step 1: Download the Proxmox VE ISO

1. Go to https://www.proxmox.com/en/downloads
2. Click **Proxmox Virtual Environment**
3. Download the latest stable ISO (e.g., `proxmox-ve_9.2-1.iso`)

### Step 2: Verify the ISO (Optional but Recommended)

Proxmox provides SHA256 checksums to verify your download isn't corrupt:

```bash
# Verify downloaded ISO (use the path of your downloaded iso)
sha256sum proxmox-ve_9.2-1.iso
```

Copy the resulting alphanumeric string and compare it to the expected checksum listed on the official website

- If the codes match exactly: Your ISO is valid, intact, and safe to use.
- If there is a mismatch: Your download is corrupted or incomplete. Delete the file and download it again

### Step 3: Write the ISO to USB

There are multiple ways to write the ISO to USB, you can use `dd` on Linux/macOS or use Rufus on Windows. 

I'm using Ventoy since it can have multiple iso at once and you can decide which iso to boot from in the boot menu, that way I can use the same USB to install Arch, Ubuntu desktop, Ubuntu server, Proxmox, TrueNAS, etc. It removes the need to have multiple USB drives for different ISOs or reflashing USB again and again when you need to install a different OS.

#### Using Ventoy (Windows/Linux) :

Ventoy lets you **copy** the ISO file to USB instead of writing raw - you can have multiple ISOs on one USB.

```bash
# 1. Download Ventoy: https://github.com/ventoy/Ventoy/releases
# 2. Install Ventoy to your USB (select device, click Install)
# 3. Copy proxmox-ve_8.2-1.iso to the USB (just drag and drop)
# 4. Boot from USB → Ventoy menu → select Proxmox ISO
```

### Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| USB not detected in BIOS | USB 3.0 port issue | Try a USB 2.0 port (black, not blue) |
| Stuck at "ISOLINUX" screen | Legacy BIOS boot on UEFI system | Change BIOS to UEFI boot mode |
| USB boots to black screen | Corrupted download or bad USB | Redownload ISO, try different USB drive |

## Phase 2 : BIOS Configuration for Proxmox

### What is BIOS/UEFI?

**BIOS** (Basic Input/Output System) is the firmware that starts when you power on the computer. It:
1. Performs POST (Power-On Self Test) — checks hardware
2. Initializes devices (RAM, drives, USB)
3. Reads boot order to find an OS to load
4. Hands control to the bootloader

**UEFI** (Unified Extensible Firmware Interface) is the modern replacement for BIOS. It supports:
- GPT partition tables (vs MBR's 2TB limit)
- Secure Boot (prevents malicious bootloaders)
- Faster boot times
- Mouse support in firmware interface

### Accessing BIOS Setup

1. Power off the host machine (shutdown completely)
2. Insert the Ventoy bootable USB
3. Power on
4. **Immediately press F10 repeatedly** (every half second) until BIOS Setup appears (your bios key might be different, refer to the internet to find out the correct key for your Motherboard/PC manufacturer)

> [!tip] HP BIOS Key
> - **F10** = Enter BIOS Setup
> - **F9** = Boot Menu (choose boot device once)
> - **Esc** = Boot options

### Required BIOS Settings

#### 1. Enable Virtualization Extensions

**Path**: `Advanced → Processor Configuration`

| Setting | Value | Why |
|---------|-------|-----|
| Intel Virtualization Technology (VT-x) | **Enabled** | Required for KVM VMs |
| Intel VT-d | **Enabled** | Required for PCI passthrough (used for passing GPU, NICs, etc to your VMs) |
| Hyper-Threading | **Enabled** | Better CPU utilization |

#### 2. Set Boot Mode to UEFI

**Path**: `Boot Options → Secure Boot Configuration`

| Setting | Value | Why |
|---------|-------|------|
| Secure Boot | **Disabled** | Proxmox doesn't support Secure Boot |
| UEFI Boot Order | **USB first** | Boot from Proxmox installer USB |

> [!tip] After Installation, Reorder
> After Proxmox is installed, you'll change the boot order to NVMe first (so it doesn't try to boot from USB). But for now, USB first.

#### 3. Disable Unnecessary Features

**Path**: `Advanced → Device Configurations`

| Setting | Value | Why |
|---------|-------|------|
| WLAN Auto Sense | **Disabled** | We use Ethernet only |
| Wake on LAN | **Disabled** | Unless you need it |
| SATA Mode | **AHCI** | Required for Proxmox to detect drives |
| Legacy Support | **Disabled** | Forces UEFI-only boot |

### Verify Settings Before Exiting

```diff
+ Intel VT-x:                       Enabled
+ Intel VT-d:                       Enabled  
+ Hyper-Threading:                  Enabled
+ Secure Boot:                      Disabled
+ UEFI Boot Order:                  USB → NVMe
+ Legacy Support:                   Disabled
+ SATA Mode:                        AHCI
```

### Save and Boot from USB

1. Save your changes and Reboot, If USB is first in boot order, you'll see:

```
Proxmox VE
Boot in graphical mode (recommended)
Boot in text mode (expert)
Advanced options
```

2. Select **Boot in graphical mode**
3. Proxmox installer should load

> [!warning] If BIOS Resets
> If the PC resets BIOS to default after power loss, the CMOS battery (CR2032) may be dead.  Replace it: pop out the coin cell on the motherboard, install a fresh one. I hope you don't have to do that :\

### Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| F10 doesn't enter BIOS | Wrong timing | Press before HP logo, or try F9, Esc |
| "Secure Boot Violation" | Secure Boot on | Disable Secure Boot in Boot Options |
| Virtualization option missing | CPU doesn't support it | Check if your processor supports VT-x. If missing, check BIOS version |
| Screen stays black after POST | DisplayPort not detected | Try HDMI port instead |
| USB not booting even when enabled | Legacy Support disabled for USB | Enable Legacy Support temporarily for install, disable after |
| Proxmox installer shows "No drives detected" | SATA mode is RAID, not AHCI | Change SATA Mode from RAID to AHCI in Advanced |

## Phase 3 : Installing Proxmox VE

### Prerequisites

- [ ] BIOS configured correctly (VT-x enabled, UEFI mode, USB boot first)
- [ ] Bootable Proxmox USB inserted
- [ ] Monitor + keyboard connected to PC
- [ ] Ethernet cable connected (DHCP from ISP router)

### Booting the Installer

1. Power on the PC
2. If USB is first in boot order → Proxmox installer menu appears
3. If not → Press boot menu key at boot → Select USB drive from boot menu
4. Select **"Install Proxmox VE"** (graphical mode)
5. Wait while the kernel loads

### Step-by-Step Installation

#### EULA Screen

- Read the End User License Agreement
- Select **"I agree"**

#### Target Disk Selection

```
Available disks:
[x] /dev/nvme0n1 - 512 GB (NVMe)    ← Select your NVMe drive or the drive you want to install the OS on (I recommend NVME for fater boot times, and better performance)
[ ] /dev/sda - 16 GB (USB drive)    ← DO NOT select USB
```

**Filesystem option**: Keep the default **ext4**

#### Location and Timezone

| Field | Value | Why |
|-------|-------|-----|
| Country | **India** | Sets timezone defaults |
| Time Zone | **Asia/Kolkata** (or your local) | Correct time for logs and cron |
| Keyboard Layout | **English (US)** | Standard layout for servers |


#### Administrator Password

| Field | Value |
|-------|-------|
| Password | **Choose a strong password** |
| Confirm | Re-enter the same password |

> [!warning] Don't Forget This Password
> This is the **root** password for the Proxmox host. You'll need it for:
> - SSH access
> - Web UI login
> - sudo/root commands

#### Management Network Configuration

```
IP address (CIDR):  192.168.1.10/24      ← Static IP (adjust for your network)
Gateway:            192.168.1.1           ← Your ISP router's IP
DNS server:         192.168.1.1           ← Use router as DNS (I'll install Pi-hole later)
Hostname:           pve.lab               ← Node hostname
```

#### Confirm and Install

- Review all settings
- Click **Install**
- The installer will:
  1. Partition the NVMe drive (boot, root, swap)
  2. Format with ext4
  3. Install Debian base system
  4. Install Proxmox packages
  5. Install GRUB bootloader
  6. Configure network

**Wait ~5-10 minutes** for installation to complete.

#### Reboot

1. When installation completes → **Reboot**
2. Remove the USB drive when prompted (or when system shuts down)
3. System boots from NVMe into Proxmox


### First Login

After reboot, you'll see the Proxmox login prompt:

```
Proxmox VE 8.2, kernel 6.5.x

pve login: root
Password: [your password]
```

Or open a web browser from your laptop and go to:

```
https://<proxmox-ip>:8006
```

> [!warning] SSL Warning
> Your browser will show "Your connection is not private" or "Invalid certificate." This is normal — Proxmox uses a self-signed certificate. Click **Advanced and Proceed to unsafe**.

Login with:
- **Username**: `root`
- **Password**: (the password you set during install)

### Post-Install: Verify Key Services

```bash
# Check the Proxmox web UI is listening
ss -tlnp | grep 8006
▶ LISTEN 0 1024 0.0.0.0:8006 0.0.0.0:* users:(("pveproxy",pid=1234))

# Check the Proxmox API
ss -tlnp | grep 8006

# Check SSH is running
ss -tlnp | grep 22

# Check DNS resolution works
host google.com
```

### Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| "No root filesystem" after reboot | Wrong boot order | F9 at boot → Select NVMe drive manually |
| Web UI unreachable | Wrong IP or firewall | `curl https://127.0.0.1:8006` from Proxmox shell to test local |
| "pveproxy: no listening sockets" | Bind error | `systemctl restart pveproxy` |
| Can't login to web UI | wrong password | Reset root password: reboot → GRUB → single user mode |
| Package errors (apt) | No internet | Check Ethernet cable, DHCP lease |

## Phase 4 : Post-Install Proxmox Configuration

### Prerequisites

- [ ] Proxmox installed and accessible via web UI
- [ ] Root password
- [ ] Internet connectivity

### Run the Post-Install Script

This script provides options for managing Proxmox VE repositories, including disabling the Enterprise Repo, adding or correcting PVE sources, enabling the No-Subscription Repo, adding the test Repo, disabling the subscription nag, updating Proxmox VE, and rebooting the system.

You can find it here : [PVE Post Install](https://community-scripts.org/scripts/post-pve-install)

What this Script Does :

- Remove Enterprise Repository : Proxmox's default repo requires a subscription. We use the **no-subscription** repo (free, same packages, just a different source).

- Remove Subscription Popup : The "No valid subscription" popup appears on every login. This disable it.

- System Update :
```bash
# Update package lists from new repos
apt update

# Upgrade all packages
apt dist-upgrade -y

# Clean up unnecessary packages
apt autoremove -y

# Reboot to apply new kernel (if one was installed)
reboot
```
> [!warning] Running Random Scripts from the Internet :)
> I have used this script multiple times but everytime I go through the code to verify it's not doing anything malicious, you should also do the same, go through all the scripts you run on your machine and if you see anything sketchy don't run it, you can easily do the same things the script is doing by running a few commands yourself.

## References

- [Proxmox Download Page](https://www.proxmox.com/downloads)
- [Official Proxmox Installation Guide](https://pve.proxmox.com/wiki/Installation)
- [Proxmox Network Configuration](https://pve.proxmox.com/wiki/Network_Configuration)
- [Proxmox No-Subscription Repo](https://pve.proxmox.com/wiki/Package_Repositories)