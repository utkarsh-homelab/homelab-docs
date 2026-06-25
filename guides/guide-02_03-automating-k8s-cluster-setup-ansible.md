# 02-03: Automating Kubernetes Setup using Ansible

## What does this automation do?

The [kubernetes guide](./guide-02_01-creating-kubernetes-cluster.md) walks through creating VMs by cloning a template in the Proxmox UI, then SSHing into each node to run ~20 commands. In the [previous guide](./guide-02_02-automating-k8s-vm-creation-terraform.md) we automated VM provisioning, in this guide we'll be automating the ~20 commands for cluster setup on the provisioned VMs we had to do manually using Ansible.

| Step | Manual (guide) | Automated (this repo) |
|---|---|---|
| Create VMs | Right-click clone in Proxmox UI | `terraform apply` refer to [previous guide](./guide-02_02-automating-k8s-vm-creation-terraform.md) for details|
| Install containerd | SSH + apt install | Ansible `common` role |
| Configure cgroup driver | Edit config.toml manually | Ansible `common` role |
| Disable swap + sysctl | 4 commands per node | Ansible `common` role |
| Install kubeadm/kubelet | apt install + hold | Ansible `kubeadm` role |
| kubeadm init + join | Two separate SSH sessions | Ansible playbook, automated token sharing |
| Install CNI | kubectl apply -f URL | Ansible `cni` role — one var to swap plugins |

The key design philosophy: **Terraform handles infrastructure (the VMs), Ansible handles configuration (the cluster).** Each tool does what it does best.

```
Terraform  ──▶  Proxmox API  ──▶  VMs created
     │
     │ (outputs MAC addresses)
     │
     ▼
DHCP reservation (manual step on your router)
     │
     │ (we fill IPs into Ansible inventory)
     │
     ▼
Ansible  ──▶  SSH into VMs  ──▶  Cluster configured
```

The Code for the automation we will be discussing can be found here: [https://github.com/utkarsh-homelab/homelab-iac#](https://github.com/utkarsh-homelab/homelab-iac#)

---

## Prerequisites

Before using this automation, ensure you have:

### On your laptop (control node)

| Requirement | Why |
|---|---|
| Ansible >= 2.15 | Configuration management |
| `community.general` collection | `modprobe` module in common role — `ansible-galaxy collection install community.general` |
| SSH key pair | Injected into VMs via cloud-init — Ansible uses the private key |

### On your network

| Requirement | Why |
|---|---|
| DHCP server with reservation support | VMs get IPs via DHCP; you reserve them by MAC |
| DNS (optional but recommended) | So nodes can resolve each other by hostname |

---

## Ansible: Configuring the Cluster

### Inventory Design

The inventory uses **group-based** organization:

```yaml
all:
  children:
    control_plane:
      hosts:
        k8s-master-1:
          ansible_host: 192.168.0.170
    workers:
      hosts:
        k8s-worker-1:
          ansible_host: 192.168.0.171
  vars:
    ansible_user: ubuntu
    ansible_become: true
```

**Why groups?** Ansible targets groups (`hosts: control_plane`, `hosts: workers`) rather than individual hosts. This means:
- Different plays can target different groups (init on control_plane, join on workers)
- Adding a new node means adding it to the right group — the plays automatically include it
- You can use `groups['control_plane'][0]` to reference the first master from any play

**Why `ansible_become: true` globally?** Every task in every role uses `become: true`. Rather than repeating it in every role's tasks, we set it globally. All configuration requires root access (installing packages, modifying system files).

### Group Variables

```yaml
kubernetes_version: "1.36"
pod_network_cidr: "10.244.0.0/16"
cni_plugin: "flannel"
show_kubeconfig_at_end: true
```

These variables are available to every playbook and role. They define:

- **`kubernetes_version`**: Controls which apt repository is added and which kubeadm/kubelet/kubectl versions are installed. This is a single source of truth for the K8s version across all nodes.

- **`pod_network_cidr`**: Passed to `kubeadm init --pod-network-cidr`. Must match the CIDR expected by your CNI plugin. Flannel defaults to `10.244.0.0/16`; Cilium can use any private range.

- **`cni_plugin`**: The CNI role reads this variable to decide which task file to include. Change this to `"cilium"` and re-run the playbook to swap CNI. 

> I will be using flannel in the cluster for now, I haven't tested cilium fully so you might have to debug if any issues arise during or after the deployment :)

- **`show_kubeconfig_at_end`**: You can set it to `true` to print the kubeconfig after the cluster has been provisioned.

### The common Role

This role performs the pre-flight configuration that Kubernetes requires on every node. It corresponds to Steps 2–4 of your manual guide.

**kernel.yaml (tasks/kernel.yaml)**

```
Disable swap       ──▶  swapoff -a + comment out /etc/fstab swap line
Load br_netfilter  ──▶  modprobe br_netfilter + persist to /etc/modules-load.d/k8s.conf
Apply sysctl       ──▶  net.bridge.bridge-nf-call-iptables = 1
                        net.bridge.bridge-nf-call-ip6tables = 1
                        net.ipv4.ip_forward = 1
```

**Why `sysctl` with `sysctl_file`?** The Ansible `sysctl` module applies the value immediately (runtime) **and** writes it to the specified file for persistence across reboots. This replaces two manual commands (`sysctl --system` + editing `sysctl.d/k8s.conf`).

**Why no reboot?** Your manual guide includes a reboot step to ensure changes survive. Ansible skips this because:
- `sysctl` with `reload: true` applies changes immediately
- `modprobe` loads modules immediately
- `swapoff` disables swap immediately
- The sysctl file and modules-load file persist across reboots anyway

A reboot would only verify that everything survives a restart — which you can do at any time without Ansible.

**containerd.yaml (tasks/containerd.yaml)**

```
apt install containerd           ──▶  Install containerd package
containerd config default        ──▶  Generate default TOML config
Replace SystemdCgroup → true    ──▶  Enable cgroup v2 driver
systemctl restart containerd    ──▶  Apply config changes
```

**Why `SystemdCgroup = true`?** Kubernetes (via kubelet) uses the systemd cgroup driver for resource management. The containerd runtime must use the same driver. Your Ubuntu 26.04 system uses cgroup v2, which requires the systemd driver.

**Why `daemon_reload: true`?** The `restart` action already triggers a daemon-reload, but specifying it explicitly handles edge cases where systemd unit files might have changed.

### The kubeadm Role

This role installs the Kubernetes tooling on every node.

```
Add K8s apt repo        ──▶  kubernetes.list with signed-by
apt install kubelet     ──▶  Installed from the repo
       kubeadm
       kubectl
apt-mark hold           ──▶  Pins versions, prevents accidental upgrades
systemctl enable kubelet ──▶  Starts kubelet (will crashloop until init — this is normal)
```

**Why `apt-mark hold`?** You don't want `apt upgrade` to inadvertently upgrade kubeadm/kubelet/kubectl. A Kubernetes upgrade is a deliberate, tested process — not something that happens in the background.

**Why enable kubelet before kubeadm init?** This is safe and expected. The kubelet will start but can't do anything until `kubeadm init` creates the cluster. It may show "waiting for kubelet to be ready" or even crashloop — this is normal and resolves after init.

### The cni Role — Pluggable by Design

```
roles/cni/tasks/
├── main.yaml        # Reads cni_plugin var, includes the right file
├── flannel.yaml     # Deploy kube-flannel.yml
└── cilium.yaml      # Install Cilium CLI + cilium install
```

**main.yaml:**
```yaml
- name: Include tasks for the selected CNI plugin
  include_tasks: "{{ cni_plugin }}.yaml"
```

That's it. The variable `cni_plugin` (from `group_vars/all.yaml`) determines which task file is included.

**Why this design?**

- To switch from Flannel to Cilium, you change **one variable** - `cni_plugin: cilium`
- To add a third CNI (e.g., Calico), you create `calico.yaml` in the same directory
- Each CNI plugin is self-contained; modifying one doesn't affect the others
- The `cni_plugin` variable acts as a "plugin registry" - set it and go

**Flannel implementation:**

```yaml
- name: Deploy Flannel overlay network
  environment:
    KUBECONFIG: /etc/kubernetes/admin.conf
  shell: |
    kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

This matches our manual command exactly. The `KUBECONFIG` environment variable tells kubectl where to find the cluster admin credentials (which were set up in the previous play).

**Cilium implementation:**

```yaml
- name: Download Cilium CLI
  get_url:
    url: https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
    dest: /tmp/cilium.tar.gz

- name: Extract Cilium CLI
  unarchive:
    src: /tmp/cilium.tar.gz
    dest: /usr/local/bin/

- name: Install Cilium
  environment:
    KUBECONFIG: /etc/kubernetes/admin.conf
  shell: |
    cilium install --wait
```

Cilium uses its own CLI (`cilium install`) rather than raw YAML. The CLI handles Helm chart download, CRD installation, and waits for daemonset readiness. The `--wait` flag blocks until all Cilium pods are healthy.

### Token Distribution Between Nodes

This is the most interesting part of the automation. In your manual guide, you copy the join command from the kubeadm init output and paste it on worker nodes. Ansible automates this with a clever pattern.

**Step 1 — Control plane generates the token:**

```yaml
- name: Generate cluster join command
  shell: kubeadm token create --print-join-command
  register: join_command_raw

- name: Set fact with join command
  set_fact:
    kubeadm_join_command: "{{ join_command_raw.stdout }}"
```

`kubeadm token create --print-join-command` generates a new token (valid 24h) and prints a ready-to-use join command that includes the token, control-plane endpoint, and CA cert hash. By `register`-ing the output and `set_fact`-ing it, we store the command in the control-plane node's hostvars.

**Step 2 — Workers read the token from the control-plane's hostvars:**

```yaml
- name: Join worker to cluster
  shell: "{{ hostvars[groups['control_plane'][0]]['kubeadm_join_command'] }}"
  args:
    creates: /etc/kubernetes/kubelet.conf
```

`hostvars[groups['control_plane'][0]]` gets the hostvars of the first control-plane node. We then read the `kubeadm_join_command` fact that we stored there.

`args: creates: /etc/kubernetes/kubelet.conf` is an idempotency guard. If the worker has already joined (indicated by the existence of `kubelet.conf`), this task skips entirely. This means re-running the playbook won't re-join already-joined nodes.

**Why this works across plays:** Ansible's `hostvars` are available across the entire playbook run, even across different plays targeting different hosts. The control-plane play runs first (storing the fact), then the worker play reads it.

### Getting the kubeconfig

If you want the kubeconfig for the cluster you can set the `show_kubeconfig_at_end` in `ansible/inventory/group_vars/all.yaml` file to `true` that will print the Kubeconfig after setting up the cluster.

You can then copy it to your machine in the `/home/{{user}}/.kube/config` file to access the cluster remotely from your machine (given your machine can connect to the server endpoint, i.e. the control-plane node)

---

## End-to-End Workflow

Here's exactly how you go from nothing to a working Kubernetes cluster.

### Phase 1: Provision VMs (if haven't already)

```bash
# 1. Navigate to Terraform directory
cd terraform/

# 2. Copy the example var files
cp terraform.tfvars.example terraform.tfvars
cp secrets.auto.tfvars.example secrets.auto.tfvars

# 3. Edit the secrets file (gitignored — safe on disk)
vim secrets.auto.tfvars
# Set your Proxmox API token

# 4. Edit the non-secret config
vim terraform.tfvars
# Set endpoint, VM definitions, resources, etc.

# 5. Initialize Terraform (downloads provider plugin)
terraform init

# 6. Preview what will be created
terraform plan

# 7. Create the VMs
terraform apply
# Output shows MAC addresses for each VM
```

### Phase 2: DHCP Reservation

```bash
# 1. From the terraform output, note the MAC addresses:
#    k8s-master-1 → 12:34:56:78:9A:BC
#    k8s-worker-1 → DE:F0:12:34:56:78

# 2. Log into your router's admin panel
# 3. Create DHCP reservations:
#    12:34:56:78:9A:BC → 192.168.0.170 (k8s-master-1)
#    DE:F0:12:34:56:78 → 192.168.0.171 (k8s-worker-1)

# 4. Verify IPs are assigned:
#    (Wait a minute for DHCP lease renewal)
#    ssh ubuntu@192.168.0.170   # Should connect
#    ssh ubuntu@192.168.0.171   # Should connect
```

### Phase 3: Configure Cluster

```bash
# 1. Navigate to Ansible directory
cd ansible/

# 2. Fill in the inventory with actual IPs
vim inventory/hosts.yaml
# Set ansible_host for each node to the reserved IPs

# 3. Verify connectivity
ansible all -m ping

# 4. Run the full cluster bootstrap
ansible-playbook playbooks/cluster.yaml
# This runs:
#   Phase 1: common (containerd, swap, sysctl) on all nodes   (~2 min)
#   Phase 2: kubeadm install on all nodes                      (~1 min)
#   Phase 3: kubeadm init + CNI on control-plane               (~3 min)
#   Phase 4: kubeadm join on workers                           (~1 min)

# 5. Verify the cluster
ssh ubuntu@192.168.0.170
kubectl get nodes
# NAME            STATUS   ROLES           AGE
# k8s-master-1    Ready    control-plane   5m
# k8s-worker-1    Ready    worker          2m
```

### (optional) Phase 4: Save the Kubeconfig

given that the `show_kubeconfig_at_end` variable was set to `true`, you can copy the kubeconfig from the ansible output to your local machine.

> Congratulations we now have a working cluster provisioned fully automated on Proxmox using Ansible and Terraform!!

> Happy tinkering with Kubernetes !!

---

## Customization Guide

### Adding More VMs

**Terraform side:** Add a new entry to the `vms` map in `terraform.tfvars`:

```hcl
vms = {
  k8s-master-1 = { ... }       # existing
  k8s-worker-1 = { ... }       # existing
  k8s-worker-2 = {             # new
    target_node = "pve2"
    vm_id       = 172
    name        = "k8s-worker-2"
    cores       = 2
    memory      = 4096
    disk_size   = 80
  }
}
```

Run `terraform apply` — only the new VM is created (existing ones are untouched).

**Ansible side:** Add the new node to the inventory:

```yaml
workers:
  hosts:
    k8s-worker-1:
      ansible_host: 192.168.0.171
    k8s-worker-2:
      ansible_host: 192.168.0.172   # new
```

Re-run the playbook with only the worker steps:

```bash
ansible-playbook playbooks/cluster.yaml --tags workers
```

The `creates: /etc/kubernetes/kubelet.conf` guard on the join task means existing workers are skipped.

### Switching CNI from Flannel to Cilium

1. Change the variable in `group_vars/all.yaml`:

```yaml
cni_plugin: "cilium"  # was "flannel"
```

2. Re-run the playbook with only the CNI tag:

```bash
ansible-playbook playbooks/cluster.yaml --tags cni
```

This runs on the control-plane node and installs Cilium.

**Important:** When switching CNI on an already-running cluster, you should also remove the old CNI. The Flannel plugin needs to be cleaned up:

```bash
ssh ubuntu@192.168.0.150
kubectl delete -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Then Cilium will take over. If you're building the cluster fresh, the playbook handles everything in order.

### Adding a New CNI Plugin

1. Create a new task file: `ansible/roles/cni/tasks/calico.yaml`:

```yaml
- name: Deploy Calico
  environment:
    KUBECONFIG: /etc/kubernetes/admin.conf
  shell: |
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
  register: calico_result
```

2. Change the variable: `cni_plugin: "calico"`

3. Run: `ansible-playbook playbooks/cluster.yaml --tags cni`

The `include_tasks: "{{ cni_plugin }}.yaml"` in `main.yaml` automatically picks up the new file. No playbook changes needed.

### Changing Kubernetes Version

1. Update the variable in `group_vars/all.yaml`:

```yaml
kubernetes_version: "1.37"
```

2. Re-run the kubeadm phase:

```bash
ansible-playbook playbooks/cluster.yaml --tags kubeadm
```

This adds the new repo and installs the new version. Note that upgrading an existing cluster requires `kubeadm upgrade` on the control-plane, which is more involved — see the [kubeadm upgrade docs](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).

### HA Control Plane (3+ Masters)

**Terraform side:** Add more masters:

```hcl
vms = {
  k8s-master-1 = { target_node = "pve1", vm_id = 170, name = "k8s-master-1", ... }
  k8s-master-2 = { target_node = "pve2", vm_id = 180, name = "k8s-master-2", ... }
  k8s-master-3 = { target_node = "pve3", vm_id = 190, name = "k8s-master-3", ... }
  k8s-worker-1 = { ... }
}
```

**Ansible side:** Add to inventory:

```yaml
control_plane:
  hosts:
    k8s-master-1:
      ansible_host: 192.168.0.170
    k8s-master-2:
      ansible_host: 192.168.0.180
    k8s-master-3:
      ansible_host: 192.168.0.190
```

**The playbook already handles this!** Look at Phase 3 - it runs on all `control_plane` hosts. The first master initializes the cluster, and additional masters would join as control-plane nodes.

One thing to adjust: the kubeadm init command should use a load-balanced endpoint (e.g., a static IP that points to all masters via keepalived or a load balancer). You'd set `--control-plane-endpoint=<LB_IP>` and add `--upload-certs` to share certificates between masters. This is a more advanced setup beyond the current scope, but the Ansible role structure supports it.

---

## Extending the Automation

### Adding a New Role

Suppose you want to add a monitoring agent (e.g., Prometheus Node Exporter) to every node:

1. Create the role structure:

```
ansible/roles/node_exporter/
└── tasks/
    └── main.yaml
```

2. Write the tasks:

```yaml
# tasks/main.yaml
- name: Install node_exporter
  apt:
    name: prometheus-node-exporter
    state: present

- name: Ensure node_exporter is running
  systemd:
    name: prometheus-node-exporter
    state: started
    enabled: true
```

3. Add the role to the `cluster.yaml` playbook's first play:

```yaml
- name: "Phase 1 — Common setup (all nodes)"
  hosts: all
  become: true
  tags: [common]
  roles:
    - common
    - node_exporter    # new role
```

### Adding a New Playbook

For future components (MetalLB, cert-manager, etc.), create separate playbooks like (I'm not sure if I'll use ansible for deploying components or via ArgoCD but you can choose if you want ansible to do it):

```
playbooks/
├── cluster.yaml       # Cluster creation
├── argocd.yaml        # ArgoCD bootstrap
├── metallb.yaml       # MetalLB (future)
├── cert-manager.yaml  # cert-manager (future)
└── traefik.yaml       # Traefik (future)
```

---

## Troubleshooting

### Ansible

| Issue | Likely Cause | Fix |
|---|---|---|
| `UNREACHABLE!` → SSH connection failed | Wrong IP in inventory | Verify IP with `ping <ip>` from your laptop; check DHCP reservation |
| `Permission denied (publickey)` | SSH key not on the VM | Verify `ssh_public_key` in Terraform was correct; run `terraform taint` on the VM and re-apply |
| `kubeadm init` fails | Containerd not running | Check `systemctl status containerd`; re-run `ansible-playbook --tags common` |
| `kubeadm join` fails: "token is invalid" | Token expired (>24h) | Re-run the playbook — a fresh token is generated |
| CoreDNS pods stuck in `Pending` | CNI not installed | Run `ansible-playbook --tags cni` |
| CoreDNS pods crashlooping | Network plugin conflict | If switching CNI, ensure old CNI is cleaned up first |
| `kubectl` not found on control-plane after playbook | `kubeadm install` failed | Check `journalctl -u kubelet`; ensure apt repo was added correctly |

### General

| Issue | Likely Cause | Fix |
|---|---|---|
| VM powers on but no network | DHCP server not responding or MAC not recognized | Check router's DHCP lease table; ensure bridge is correct in `terraform.tfvars` |
| `kubectl get nodes` shows `NotReady` | CNI not deployed or containerd not healthy | `kubectl describe node` for conditions; `journalctl -u kubelet` for errors |
| Worker node shows `NotReady` but looks fine | Network between nodes (firewall/iptables) | Check `sysctl net.bridge.bridge-nf-call-iptables` returns 1; verify pods on worker can reach control-plane |

---

> In the future I'll probably setup Github Actions to run these automations on [GitHub-hosted runners](https://docs.github.com/en/actions/concepts/runners/github-hosted-runners) on the proxmox cluster, so whenever you push the changes to github, it automatically gets reflected in the cluster, for now I'll be running the automations myself from my personal computer (I don't want to overwhelm my proxmox host, I have other plans that I think will be more fun use of the compute 😇.)


## References

- [Ansible Official](https://docs.ansible.com/)
- [Ansible Documentation](https://docs.ansible.com/projects/ansible/latest/index.html)
- [kubeadm installation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [kubeadm cluster creation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Flannel](https://github.com/flannel-io/flannel)
- [Cilium](https://docs.cilium.io/en/stable/)