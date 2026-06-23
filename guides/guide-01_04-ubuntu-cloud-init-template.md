## 01-04: Create Cloud-Init Template (Ubuntu 26.04)

## What is Cloud-Init?

Cloud-Init is the standard way to bootstrap cloud VMs (used by AWS, GCP, Azure). We'll create a template VM that Ansible and Terraform will clone for our K8s nodes.

> What is Cloud-Init?
> Cloud-Init is a package that runs at first boot to configure:
> - Hostname
> - Network (static IP, DNS)
> - SSH keys
> - Users, passwords
> - Disk resizing
> - First-boot scripts
>
> This is the same technology cloud providers use. When you launch an EC2 instance with a user-data script, Cloud-Init runs it.

## Step 1: Install required tools on the Proxmox node

```bash
apt update
apt install -y libguestfs-tools wget
```
`libguestfs-tools` gives you `virt-customize`, which is a straightforward way to inject `qemu-guest-agent` into the cloud image before importing it into Proxmox.

## Step 2: Download Ubuntu Cloud Image

### Through GUI

- Choose an image from [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/) 

- I'll be using [ubuntu-26.04-server-cloudimg-amd64.img](https://cloud-images.ubuntu.com/releases/resolute/release/ubuntu-26.04-server-cloudimg-amd64.img)

- In the Proxmox UI go to local > ISO images > Download from URL 

- Enter the Image URL > click Query URL > Download (you can also add the checksum in advanced options to verify the image)

### Through CLI

```bash
cd /var/lib/vz/template/iso
wget -O ubuntu-26.04-server-cloudimg-amd64.img https://cloud-images.ubuntu.com/releases/resolute/release/ubuntu-26.04-server-cloudimg-amd64.img
```

## Step 3: Inject qemu-guest-agent into the image

```bash
virt-customize -a ubuntu-26.04-server-cloudimg-amd64.img --install qemu-guest-agent
```
This bakes the package into the image itself, so every clone made from the template already has the guest agent installed without depending on first-boot package installation.

There's a small problem here I noticed during testing - 
the virt-customize populates the machine-id so every clone will end up having the same machine ID, that's not good, it creates issues with proxmox trying to differentiate the machines, might cause issues with k8s as well, idk.

So one way to fix it is to run the commands I have listed at the end to remove the machine ID so after reboot the VMs will get new hopefully unique machine ID

or use this command instead

```bash
virt-customize \
  -a ubuntu-26.04-server-cloudimg-amd64.img \
  --install qemu-guest-agent \
  --truncate /etc/machine-id
```

> QEMU Guest Agent
> The QEMU guest agent runs inside the VM and communicates with Proxmox. It enables:
> - Proper VM shutdown (instead of ACPI power-off)
> - IP address display in the Proxmox UI
> - Filesystem freeze before snapshots (consistent backups)
>
> Without it, Proxmox treats the VM like a physical machine — no graceful shutdown, no IP visibility.

## Step 4: Create the VM shell in Proxmox

```bash
qm create 9000 --name ubuntu-26-04-cloudinit --memory 4096 --cores 2 --cpu host --net0 virtio,bridge=vmbr0
```

This creates the Proxmox VM definition that will become your template.

## Step 5: Import the cloud image as the main disk

```bash
qm importdisk 9000 /var/lib/vz/template/iso/ubuntu-26.04-server-cloudimg-amd64.img local-lvm
```

This imports the downloaded cloud image into your target storage so Proxmox can attach it as the VM’s root disk.

## Step 6: Attach the imported disk and set boot options

```bash
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --boot c --bootdisk scsi0
```

## Step 7: Add the cloud-init drive

```bash
qm set 9000 --ide2 local-lvm:cloudinit
```

A Proxmox VM needs an attached cloud-init drive for Proxmox to pass cloud-init metadata like user, SSH key, DNS, and DHCP/static network settings to clones.

## Step 8: Configure serial console and guest agent

```bash
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1
```

Serial console settings are commonly recommended for cloud images, and the Proxmox VM config must also enable the QEMU guest agent feature so Proxmox can talk to the agent inside the guest after boot. Enabling the Proxmox-side agent flag alone is not enough, which is why you installed the package inside the image earlier.

## Step 9: Set template cloud-init defaults

```bash
qm set 9000 --ciuser ubuntu
qm set 9000 --ipconfig0 ip=dhcp
qm set 9000 --sshkeys ~/.ssh/authorized_keys
```

Additionally if you have followed the guide for setting up Pihole you can set that as the DNS server for the template (I will be doing it)

```bash
qm set 9000 --nameserver 192.168.0.101
```

## Step 10: Optionally resize the base disk

```bash
qm resize 9000 scsi0 50G
```

If the cloud image root disk is too small for your general-purpose template, resize it before converting the VM to a template.

## Step 11: Convert it to a template

```bash
qm template 9000
```

At this point the VM becomes a reusable Proxmox template
I'll use this template later with Terraform to clone into your control-plane and worker nodes :)

> Why Cloud-Init over manual VM setup?
> Manual setup is fine for one VM. With Cloud-Init, you can:
> - Spin up a new VM in 30 seconds
> - Clone for testing without manual reconfiguration
> - Define everything in code (Terraform + Cloud-Init)
> - Rebuild from scratch in minutes if something breaks

## Step 12: Test the Template

```bash
# Create a test VM from the template
qm clone 9000 999 --name test-vm --full
qm set 999 --ipconfig0 ip=192.168.0.99/24,gw=192.168.0.1 --ciupgrade=0
qm start 999

# Wait 60 seconds, then SSH in
ssh ubuntu@192.168.1.99
```
If you can SSH in, the template works. Delete the test VM:

```bash
qm stop 999 && qm destroy 999
```

### Fix - clones with same machine id

If you need to reset your machine-id, log into the vm and run the following

```bash
sudo rm -f /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
```

Then shut it down and do not boot it up.A new id will be generated the next time it boots.If it does not you can run:

```bash
sudo systemd-machine-id-setup
```

### Fix - DNS reconfigured by router

Your qm cloudinit dump is only the generated cloud-init metadata; Ubuntu still applies its own netplan/systemd-resolved logic after boot, and DHCP can override the resolver choice. Multiple Proxmox/Ubuntu reports show that DNS settings may look correct in cloud-init but the guest continues using the DHCP-provided DNS unless the network config explicitly sets nameservers in netplan.

I was seeing the same issue on my VMs, here's how to fix it:

Keep DHCP for the address, but make the guest’s netplan explicitly define the DNS server too:

```text
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      nameservers:
        addresses: [192.168.0.101]
        search: [lab.uttutu]
```

Then run:

```bash
sudo netplan apply
sudo reboot
```

We can also try and add this as a custom snippet in the template so it doesn't happen again, but since I'm planning to replace my ISP router with an custom OpnSense router in the future, then the DHCP from that router will hand out Pihole as the DNS, so I won't be automating this. 

If your router allows to change the DNS server you can change the DNS there to point to Pihole mine doesn't unfortunately so I can't do it :(

## References

- [Cloud-init documentation](https://docs.cloud-init.io/en/latest/)
- [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/)
- [Virt-customize](https://libguestfs.org/virt-customize.1.html)
- [Qemu-guest-agent](https://pve.proxmox.com/wiki/Qemu-guest-agent)
- [Proxmox qm docs](https://pve.proxmox.com/pve-docs/qm.1.html)