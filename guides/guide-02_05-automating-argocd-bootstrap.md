# 02-05: ArgoCD Bootstrap Automation using Ansible

> [!WARNING]
> I will be deploying other applications through ArgoCD in the future so the main branch will have additional apps that might break if their requirements are not met.
>
> for example -
>
> e.g. 1: The MetalLB app needs a ip-block outside of the DHCP range of your router, you'll have to configure this according to your network setup.
>
> e.g. 2: The CSI-Driver-NFS app needs a nfs-share for persistent storage, I'll be using a nfs-share on proxmox for now, in future I'll use a dedicated NAS, so you'll have to configure this according to your storage setup.
>
> I would recommend using the [bootstrap branch](https://github.com/utkarsh-homelab/homelab-gitops/tree/bootstrap) instead, since it only has the files required for the bootstrap setup, i.e. the app-of-apps `root` application, `argocd` application and the `argocd-install.yaml` file.

## Overview

This automation mirrors the manual steps from [previous guide](./guide-02_04-argocd-bootstrap-and-gitops-setup-manual.md) but executes them as a single Ansible playbook. It handles:

- Installing ArgoCD CRDs from upstream URLs
- Generating bootstrap manifests via Helm template (or using a vendored copy)
- Applying manifests with server-side apply
- Waiting for ArgoCD to be ready
- Printing the initial admin password
- Optionally applying the root App of Apps

**Correspondence with the manual guide:**

| Manual Step | Ansible Action |
|---|---|
| Step 3: Install CRDs | `kubectl apply/create` from upstream URLs |
| Step 4: Bootstrap ArgoCD | `kubectl apply --server-side -f argocd-install.yaml` |
| Step 4: Wait for ready | `kubectl wait --for=condition=ready pod` |
| Step 5: Apply root App of Apps | Optional task (tag: `root-app`) |
| Get admin password | `kubectl get secret ... \| base64 -d` |

---

## Prerequisites

### Required

- [ ] **Kubernetes cluster** must be running and healthy.
- [ ] kubectl configured on your machine (kubeconfig at `~/.kube/config`).

### For initial run (manifest generation)

- **Helm binary** on the Ansible control node (your laptop), OR a pre-generated `argocd-install.yaml` in `roles/argocd/files/`

> We won't be using the `argocd-install.yaml` bootstrap file from the `homelab-gitops` repo, we'll be regnerating it with the automation to remove dependecy and to keep the flow fully automated, you can still use that file if you want to deploy argocd manually.

### For `--tags root-app`

- **homelab-gitops repo** must exist on GitHub with the `apps/root.yaml` file (I'll keep the repo public so you don't have to worry about this 😇)
- The control-plane node must have **internet access** to download the raw manifest from GitHub

---

## The CRD Problem - Why Separate Installation?

Kubernetes has two limits that affect ArgoCD's bootstrap:

### The 256KB annotation limit

When you use `kubectl apply` (or `kubectl apply --server-side`), Kubernetes stores the last-applied configuration in annotations. These annotations have a size limit of approximately 256KB. The ApplicationSets CRD, when processed through Helm, can have annotations that exceed this limit.

### The 1MB request size limit

Kuberentes API server has a 1MB limit on request sizes. Combined with Helm's default behavior of including large annotations in rendered manifests, the total bootstrap manifest can approach or exceed this limit.

### The three-part solution

The automation uses three strategies to work around these limits:

**1. `--skip-crds` in helm template**

```bash
helm template argocd argo/argo-cd \
  --skip-crds \
  --values values.yaml \
  > argocd-install.yaml
```

This excludes all CRD manifests from the Helm output. CRDs are the largest resources in the ArgoCD chart. By excluding them, the bootstrap YAML stays well under the size limits.

**2. Separate CRD installation from upstream URLs**

```yaml
- name: Install Application CRD
  shell: |
    kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/{{ argocd_version }}/manifests/crds/application-crd.yaml

- name: Install ApplicationSet CRD
  shell: |
    kubectl create -f https://raw.githubusercontent.com/argoproj/argo-cd/{{ argocd_version }}/manifests/crds/applicationset-crd.yaml

- name: Install AppProject CRD
  shell: |
    kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/{{ argocd_version }}/manifests/crds/appproject-crd.yaml
```

CRDs installed directly from upstream URLs don't have Helm annotations, so they avoid the size limits entirely. The `applicationset-crd.yaml` is applied with `kubectl create` instead of `kubectl apply` because even server-side apply can't handle its annotation size - `kubectl create` doesn't add annotations.

**3. `--server-side` flag on the main apply**

```bash
kubectl apply --server-side -f /tmp/argocd-install.yaml
```

Server-side apply handles large manifests differently than client-side apply. It's more efficient with large resources and is the recommended approach for bootstrapping complex applications like ArgoCD.

---

## Step-by-Step: What the Playbook Does

### Step 1: Install CRDs (`tags: crd`)

Three tasks download and apply the ArgoCD CRDs directly from GitHub:

- **`application-crd.yaml`**: Defines the `Application` custom resource - the core building block of ArgoCD's App of Apps pattern

- **`applicationset-crd.yaml`**: Defines the `ApplicationSet` custom resource - for generating Applications from templates (used for multi-cluster or multi-environment setups, we don't use it but I'm adding it anyway if you want to use it.)

- **`appproject-crd.yaml`**: Defines the `AppProject` custom resource - for grouping Applications and controlling access

The CRD URLs are constructed from `argocd_version` variable:

```
https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.4/manifests/crds/application-crd.yaml
```

This means changing `argocd_version` in `group_vars/all.yaml` automatically points to the correct CRD URLs for that version.

### Step 2: Generate Bootstrap Manifests

Two modes:

**Mode A - Vendored (default after first generation)**

If `roles/argocd/files/argocd-install.yaml` exists, it's used directly. This file is generated once (via `helm template`) and committed to the repo. Benefits:

- No Helm dependency on the control node
- Fast execution (no download/generation time)
- Version-pinned - you know exactly which manifests will be applied

**Mode B - Runtime generation (fallback)**

If the vendored file doesn't exist, the role runs `helm template` on the Ansible control node. Before that, it downloads the Helm values from the `homelab-infra-charts` repo:

```yaml
- name: Download ArgoCD values from homelab-infra-charts
  delegate_to: localhost
  get_url:
    url: "{{ argocd_values_url }}"
    dest: /tmp/argocd-values.yaml
```

The `argocd_values_url` is defined in `group_vars/all.yaml`:

```yaml
argocd_values_url: "https://raw.githubusercontent.com/utkarsh-homelab/homelab-infra-charts/main/charts/argocd/values/prod.yaml"
```

This means the values are **not vendored** in the Ansible role - the single source of truth lives in `homelab-infra-charts/charts/argocd/values/prod.yaml`. When I update the values there, the Ansible automation picks them up on the next run without any additional changes.

The helm template command uses these flags:

| Flag | Purpose |
|---|---|
| `--version 9.7.0` | Pin chart version |
| `--namespace argocd --create-namespace` | Set target namespace |
| `--skip-crds` | Exclude CRDs (installed separately) |
| `--values /tmp/argocd-values.yaml` | Apply custom values from infra-charts repo |
| `--set argo-cd.configs.cm.url=https://argocd.uttutu.xyz` | Override URL from group_vars |

The generated manifests are written to `roles/argocd/files/argocd-install.yaml`, which then becomes the vendored file for future runs.

### Step 3: Apply Manifests

The manifests are applied with server-side apply:

```bash
kubectl apply --server-side -f /tmp/argocd-install.yaml
```

`--server-side` moves the apply logic from kubectl (client) to the API server (server). This avoids client-side size limits and provides better conflict resolution for complex resources.

### Step 4: Wait for ArgoCD Server

```bash
kubectl wait --for=condition=ready pod \
  -n argocd -l app.kubernetes.io/name=argocd-server \
  --timeout=300s
```

This waits up to 5 minutes for the ArgoCD server pod to enter the `Ready` state. If it times out, the playbook continues (graceful failure) - the user can check `kubectl get pods -n argocd` to diagnose.

### Step 5: Print Admin Password

The initial admin password is retrieved from the `argocd-initial-admin-secret` and printed to the console. This is the same command from the manual guide:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d
```

### Step 6 (Optional): Apply Root App of Apps

See [Section 7](#7-the-root-app-of-apps).

---

## Running the Automation

### Basic Bootstrap

```bash
# From the ansible/ directory
ansible-playbook playbooks/argocd.yaml
```

### Access ArgoCD

Since MetalLB and Traefik aren't deployed yet, use port-forwarding:

```bash
# In a separate terminal:
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Then open https://localhost:8080 in your browser
# Username: admin
# Password: <from playbook output>
```

### Running Specific Steps

The playbook uses Ansible tags. You can run individual steps:

| Command | What it does |
|---|---|
| `ansible-playbook playbooks/argocd.yaml` | Full bootstrap (CRDs + manifests + wait) |
| `ansible-playbook playbooks/argocd.yaml --tags crd` | Install/update CRDs only |
| `ansible-playbook playbooks/argocd.yaml --tags root-app` | Apply root App of Apps only |

---

## The Root App of Apps

### What It Is

The root App of Apps is an ArgoCD `Application` resource that points to the `apps/` directory of the `homelab-gitops` repo. When applied, ArgoCD reads the `apps/` directory and creates one `Application` resource per file. Each of those Applications syncs its respective Helm chart from the `homelab-infra-charts` repo.

This is the foundation of the GitOps self-management pattern:

```
root.yaml ──▶ discovers apps/ directory
  │
  ├── apps/argocd.yaml         ──▶ self-manages ArgoCD via homelab-infra-charts
  ├── apps/metallb.yaml        ──▶ manages MetalLB (future)
  ├── apps/traefik.yaml        ──▶ manages Traefik (future)
  ├── apps/cert-manager.yaml   ──▶ manages cert-manager (future)
  └── ... more apps
```

### When to Apply

Apply the root App of Apps **after**:

- ArgoCD is confirmed running (`kubectl get pods -n argocd` shows all pods ready)

### How to Apply

```bash
ansible-playbook playbooks/argocd.yaml --tags root-app
```

This downloads `apps/root.yaml` from the `homelab-gitops` repo and applies it with `kubectl apply -f -`.

The repo URL is configurable via the `argocd_gitops_repo` variable in `group_vars/all.yaml`:

```yaml
argocd_gitops_repo: "https://github.com/utkarsh-homelab/homelab-gitops"
```

### What Happens After

1. ArgoCD syncs the root Application
2. Root discovers the `apps/` directory in the repo
3. For each `.yaml` file in `apps/`, root creates an Application resource
4. Each Application syncs its Helm chart from `homelab-infra-charts`
5. ArgoCD manages itself via `apps/argocd.yaml`

> From this point, ArgoCD is self-managing. Changes to any component should be made by updating the appropriate chart in `homelab-infra-charts` or the Application manifest in `homelab-gitops`, not by modifying the cluster directly.

---

## Customization

### Changing the ArgoCD URL

```yaml
# group_vars/all.yaml
argocd_url: "https://argocd.your-domain.com"
```

This is injected into the Helm values via `--set argo-cd.configs.cm.url={{ argocd_url }}` during manifest generation. It affects:
- The `configs.cm.url` value in ArgoCD's ConfigMap (used for link-backs in notifications)
- The Ingress host if ingress is enabled

### Changing the ArgoCD Version

Update two variables in `group_vars/all.yaml`:

```yaml
argocd_helm_version: "9.7.0"    # Helm chart version
argocd_version: "v3.4.4"        # ArgoCD app version (used in CRD URLs)
```

To find matching versions:
- Chart versions: `helm search repo argo/argo-cd --versions`
- App version: check the chart's `appVersion` or the ArgoCD release page

The CRD URLs automatically use `argocd_version`:

```
https://raw.githubusercontent.com/argoproj/argo-cd/{{ argocd_version }}/manifests/crds/application-crd.yaml
```

After changing versions, regenerate the vendored manifests:

```bash
ansible-playbook playbooks/argocd.yaml --tags regenerate
```

### Changing the GitOps Repo URL

```yaml
# group_vars/all.yaml
argocd_gitops_repo: "https://github.com/your-org/your-gitops-repo"
```

This affects the root-app task that downloads `apps/root.yaml` from the repo.

### Adding Pre-Bootstrap Annotations

Edit `roles/argocd/files/values.yaml` to customize the Helm values. The `--set` flag is used only for the `url` - all other values come from the file.

---

## Troubleshooting

### ArgoCD server pod stays in `Pending`

**Cause:** Usually a CNI/networking issue or insufficient resources.

**Check:**
```bash
kubectl describe pod -n argocd -l app.kubernetes.io/name=argocd-server
```

**Fix:** Ensure the cluster CNI is healthy. If using Flannel, check `kubectl get pods -n kube-flannel`. Re-run the cluster playbook if needed.

### `kubectl apply --server-side` fails with "too large"

**Cause:** The bootstrap YAML exceeds the 1MB request limit, even with `--skip-crds`.

**Fix:** This is rare if `--skip-crds` is used. If it happens:
1. Check the file size: `wc -c /tmp/argocd-install.yaml`
2. Reduce the number of components installed at bootstrap (e.g., disable redis, notifications in values.yaml)
3. Split the bootstrap into multiple files

### `applicationset-crd.yaml` fails with annotation limit

**Cause:** Using `kubectl apply` instead of `kubectl create` for this CRD.

**Fix:** The playbook uses `kubectl create` for this CRD. If someone changed it to `apply`, switch back:
```yaml
shell: |
  kubectl create -f https://.../applicationset-crd.yaml 2>/dev/null || true
```

### Root app apply fails: "not found"

**Cause:** The `homelab-gitops` repo doesn't exist yet, or the `apps/root.yaml` file hasn't been committed.

**Fix:** The root-app task is tagged so it's skipped by default. Only run it when the repo is ready. If it fails:
1. Verify the repo exists: `curl -sI {{ argocd_gitops_repo }}`
2. Verify the root.yaml path: `curl -sI {{ argocd_gitops_repo }}/raw/main/apps/root.yaml`
3. Check `argocd_gitops_repo` in `group_vars/all.yaml`

### ArgoCD Ingress shows 404/default backend

**Expected behavior.** At bootstrap time, MetalLB and Traefik haven't been deployed yet. The Ingress resource exists but has no controller to process it. Use port-forwarding until those components are installed.

### `helm template` hangs or fails

**Cause:** Network issue reaching the Helm repo, or missing dependencies.

**Fix:**
```bash
# Manually test the helm template command
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm template argocd argo/argo-cd --namespace argocd --skip-crds --values roles/argocd/files/values.yaml > /tmp/test.yaml
```

If this works, the issue is in the Ansible task's env/path. If not, check internet connectivity.

---

## References

- [Manual ArgoCD Bootstrap Guide](./guide-02_04-argocd-bootstrap-and-gitops-setup-manual.md)
- [ArgoCD Helm Chart](https://artifacthub.io/packages/helm/argo/argo-cd)
- [ArgoCD Self-Managed Patterns](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps)
- [Kubernetes Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/)
