# 02-02: Automating Kubernetes Cluster VM creation on Proxmox using Terraform 

## Why Automate?

The [previous guide](./guide-02_01-creating-kubernetes-cluster.md)  walks through creating VMs by cloning a template in the Proxmox UI, then SSHing into each node to run ~20 commands. This works, but it's:

- **Repetitive** — every new node means the same 20 commands
- **Error-prone** — one missed step breaks the cluster
- **Non-reproducible** — you can't recreate the exact same cluster in one command
- **Hard to scale** — adding 5 workers means 5x the manual work

Automation solves all of that.

The Code for the automation we will be doing can be found here: [https://github.com/utkarsh-homelab/homelab-iac#](https://github.com/utkarsh-homelab/homelab-iac#)

---

## Prerequisites

Before using this automation, ensure you have:

### On your laptop (control node)

| Requirement | Why |
|---|---|
| Terraform >= 1.5 | Infrastructure provisioning |
| Proxmox API token | Terraform authenticates with Proxmox via this token |

> Proxmox API token creation
> You can create it from the UI or the command line, I'll do it using the UI
> In Proxmox UI got to Datacenter > Permissions > API token > click Add > select a `User` and a `Token ID` > uncheck `Privilege Seperation` > Click Add > the token value value will be shown only this one time make sure you save it securely.

### On Proxmox

| Requirement | Why |
|---|---|
| Ubuntu cloud-init template | VMs are cloned from this — must exist with ID matching `template_id` |
| At least one Proxmox node | VMs are scheduled on `target_node` |
| Network bridge (e.g. `vmbr0`) | VMs connect to this bridge |
| Storage (e.g. `local-lvm`) | VM disks are created on this storage |

### On your network

| Requirement | Why |
|---|---|
| DHCP server with reservation support | VMs get IPs via DHCP; we reserve them by MAC |
| DNS (optional but recommended) | So nodes can resolve each other by hostname |

---

## Terraform: Provisioning VMs on Proxmox

### Step 1: Clone IAC repo

- Clone the `homelab-iac` repo from [Github](https://github.com/utkarsh-homelab/homelab-iac)

```bash
git clone git@github.com:utkarsh-homelab/homelab-iac.git
cd homelab-iac
```

### Step 2: Setup terraform variables and secret

```bash
cd terraform

# 1. Create both files from examples
cp terraform.tfvars.example terraform.tfvars
cp secrets.auto.tfvars.example secrets.auto.tfvars

# 2. Edit with actual values
vim secrets.auto.tfvars    # add real API token and SSH key
vim terraform.tfvars       # set endpoint, VM definitions, etc.
```
### Step 3: Run terraform commands to create the VMs

```bash
terraform init          # Prepares the working directory, downloads the required providers, and sets up the backend/state configuration.
terraform plan          # Shows a preview of what Terraform will create, change, or destroy before making any real changes.
terraform apply         # Executes the plan and actually creates or updates your infrastructure.
```

### Step 4: Changing or Deleting the VMs

> If you want to delete the vms or change anything about the vms (cpu, mem, storage, etc) **don't do it trough proxmox**, that will cause a drift and the automations will fail, always go through terraform for any infra change 

- To change Infra setting, make changes to the `terraform.tfvars` file and other concerned files, and run

```bash
terraform plan
terraform apply
```

- To Delete the VMs run

```bash
terraform destroy     # Removes all infrastructure managed by that Terraform configuration.
```

---

## Terraform Design Choices

### Provider Choice: bpg/proxmox

We will use the `bpg/proxmox` provider instead of the older `Telmate/proxmox`:

| Aspect | `bpg/proxmox` | `Telmate/proxmox` |
|---|---|---|
| API | Proxmox VE API v2 (native) | SSH + partial API |
| Cloud-init | Native `initialization` block | `cicustom` workaround |
| Active maintenance | Yes (2024+) | Slower updates |
| Schema | Clean HCL5, well-documented | Legacy patterns |
| Proxmox cluster | Native support | Partial |

The provider needs the following to authenticate:

```
provider "proxmox" {
  endpoint  = var.proxmox_endpoint   # https://pve.example.com:8006/
  api_token = var.proxmox_api_token  # root@pam!my-token=xxxxxxxx
  insecure  = var.proxmox_insecure   # true if self-signed cert
}
```

### The k8s-vm Module

The module is a wrapper around `proxmox_virtual_environment_vm` that encapsulates everything needed to create a Kubernetes node:

```hcl
resource "proxmox_virtual_environment_vm" "this" {
  node_name  = var.target_node    # which Proxmox host
  vm_id      = var.vm_id          # e.g., 150
  name       = var.name           # e.g., k8s-master-1
  started    = true               # power on after creation
  on_boot    = true               # auto-start on Proxmox boot
  migrate    = true               # allow live migration

  clone {
    vm_id = var.template_id       # clone from cloud-init template
    full  = true                  # full clone (independent)
  }
```

### Variables Strategy

**Per-VM variables** are defined in a single map:

```hcl
vms = {
  k8s-master-1 = {
    target_node = "pve1"
    vm_id       = 170
    name        = "k8s-master-1"
    cores       = 2
    memory      = 4096
    disk_size   = 50
  },
  k8s-worker-1 = {
    target_node = "pve1"
    vm_id       = 171
    name        = "k8s-worker-1"
    cores       = 2
    memory      = 4096
    disk_size   = 50
  }
}
```

**Shared defaults** are top-level variables:

| Variable | Default | Why separate |
|---|---|---|
| `template_id` | `9000` | Same template for all VMs |
| `bridge` | `vmbr0` | Same network bridge |
| `storage` | `local-lvm` | Same storage |

This separation means:
- Adding a VM adds one block to the `vms` map — no need to repeat template/bridge/storage
- Changing the template for all VMs changes one variable
- Each VM specifies only what varies: node placement, resources, ID

**Secrets are split into a separate file.** The `proxmox_api_token` variable go in `secrets.auto.tfvars` (listed in `.gitignore`) rather than in `terraform.tfvars`. Terraform auto-loads all files matching `*.auto.tfvars`, so both files are merged automatically at runtime. This keeps secrets out of Git while allowing the non-secret configuration (`vms`, `bridge`, `storage`) to be version-controlled. 

### Outputs: MAC Addresses for DHCP

```hcl
output "vm_macs" {
  value = {
    for k, vm in module.k8s_nodes : k => {
      name       = vm.name
      vm_id      = vm.vm_id
      mac_address = vm.mac_address
    }
  }
}
```

After `terraform apply`, the output looks like:

```
vm_macs = {
  "k8s-master-1" = {
    name        = "k8s-master-1"
    vm_id       = 170
    mac_address = "12:34:56:78:9A:BC"
  }
  "k8s-worker-1" = {
    name        = "k8s-worker-1"
    vm_id       = 171
    mac_address = "DE:F0:12:34:56:78"
  }
}
```

You take these MACs to your router's DHCP reservation page and assign fixed IPs.

### Lifecycle Safety

```hcl
lifecycle {
  prevent_destroy = true
}
```

This is a safety net. Terraform will **refuse** to destroy any VM created by this module. To destroy a VM, you must either:
1. Remove `prevent_destroy`, apply, re-add it — intentional two-step process
2. Use `terraform state rm` to remove the resource from state (advanced, only if you know what you're doing)

This prevents accidents like `terraform destroy` wiping your entire cluster. I have commented it out in the actual code for now, since I'll be recreating this cluster multiple times for testing and tinkering.

### Secrets Management

The repo uses a **two-file variable strategy** to keep secrets out of version control:

```
terraform/
├── terraform.tfvars              # Non-secret vars — CAN be committed
├── terraform.tfvars.example      # Documented example — committed
├── secrets.auto.tfvars           # Secrets — .gitignored
└── secrets.auto.tfvars.example   # Placeholder example — committed
```

**Why split?** Terraform's variable files are just plain text. If `terraform.tfvars` contained the API token and was accidentally committed, anyone with access to the repo could manage your Proxmox infrastructure. By isolating secrets into a separate file that's explicitly gitignored, you can safely version-control the cluster configuration (VM definitions, resource specs, network settings) without exposing credentials.

**How Terraform loads variables:**
Terraform automatically loads all `*.auto.tfvars` files in the working directory alongside the explicitly specified `-var-file` (or the default `terraform.tfvars`). This means:
- `terraform.tfvars` is loaded automatically with non-secret configuration
- `secrets.auto.tfvars` is loaded automatically with secrets
- If a variable appears in both files, the last one loaded wins — so `secrets.auto.tfvars` (loaded alphabetically after `terraform.tfvars`) takes precedence

**What goes where:**

| File | Contents | Committed? |
|---|---|---|
| `terraform.tfvars` | `proxmox_endpoint`, `proxmox_insecure`, `template_id`, `bridge`, `storage`, `vms` map | Yes (gitignored but CAN be — it has no secrets) |
| `secrets.auto.tfvars` | `proxmox_api_token` | **No** — explicitly in `.gitignore` |

**Setup on first clone:**

```bash
# 1. Create both files from examples
cp terraform.tfvars.example terraform.tfvars
cp secrets.auto.tfvars.example secrets.auto.tfvars

# 2. Edit with actual values
vim secrets.auto.tfvars    # add real API token and SSH key
vim terraform.tfvars       # set endpoint, VM definitions, etc.
```

**Git safety:** The `.gitignore` already contains:
```
terraform/secrets.auto.tfvars
terraform/*.tfstate
terraform/*.tfstate.backup
```

To verify nothing secret would be committed, run `git status` from the repo root before pushing — `secrets.auto.tfvars` should not appear.

---

## Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `Error: No VM with ID 9000 found` | Template doesn't exist | Set `template_id` to your actual template ID: `terraform.tfvars` |
| `Error: 401 Unauthorized` | Invalid API token | Regenerate the token in Proxmox UI under Datacenter → Permissions → API Tokens |
| `Error: timeout waiting for condition` | Network issues or clone took too long | Re-run `terraform apply` — it's safe (prevent_destroy protects existing VMs) |
| `prevent_destroy` blocks `terraform destroy` | Safety feature | Temporarily remove `prevent_destroy` from the module, apply, then re-add |

---

## References

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [bpg/proxmox Terraform provider docs](https://registry.terraform.io/providers/bpg/proxmox/latest/docs)