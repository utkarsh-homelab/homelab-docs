# 02-04: Bootstrap ArgoCD + Self-Management (App of Apps) - Manual

## Prerequisites

- [ ] Kubernetes cluster running (kubeconfig at `~/.kube/config`)
- [ ] kubectl configured on your machine

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

## GitOps Setup

1. [utkarsh-homelab/homelab-infra-charts](https://github.com/utkarsh-homelab/homelab-infra-charts): All vendored umbrella charts.

2. [utkarsh-homelab/homelab-gitops](https://github.com/utkarsh-homelab/homelab-gitops): The App of Apps - the single source of truth that ArgoCD reads to manage itself and all other components.

## Step 1: [homelab-infra-charts](https://github.com/utkarsh-homelab/homelab-infra-charts) - Vendored chart setup

> Clone the `homelab-infra-charts` repo from [github](https://github.com/utkarsh-homelab/homelab-infra-charts)

### charts/argocd/Chart.yaml

```yaml
apiVersion: v2
name: argocd
description: Vendored ArgoCD Helm chart for uttutu homelab
type: application
version: 1.0.0
appVersion: v3.4.4
dependencies:
  - name: argo-cd
    version: 9.7.0
    repository: https://argoproj.github.io/argo-helm
```

### charts/argocd/values/prod.yaml

Values are namespaced under `argo-cd:` because the vendor chart wraps the upstream argo-cd chart as a Helm dependency. No image overrides needed - the upstream default images are used directly.

> We'll setup traefik and cert-manager in upcoming guides, for now the Ingress and certificates won't work. Also I have hardcoded my urls in the file, change them according to your setup, I'll create an automation for the bootstrap too where I'll make it a variable so you can change it 😇.

```yaml
argo-cd:
  configs:
    params:
      server.insecure: true
    cm:
      timeout.reconciliation: 60s
      url: https://argocd.uttutu.xyz
  server:
    replicas: 1
    ingress:
      enabled: true
      ingressClassName: traefik
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hostname: argocd.uttutu.xyz
      tls: true
  dex:
    enabled: false
  redis-ha:
    enabled: false
  controller:
    replicas: 1
  repoServer:
    replicas: 1
    serviceAccount:
      create: false
  applicationSet:
    replicas: 1
  notifications:
    enabled: false
```

> [!NOTE]
> I ended up setting up a wildcard certificate later and that is reflected in the main branch.
>
> So I'm updating it here at well, earlier we were using `argocd-tls` for argocd as a placeholder before I had deployed argocd.

## Step 2: [homelab-gitops](https://github.com/utkarsh-homelab/homelab-gitops) - Bootstrap + App of Apps setup

> Clone the `homelab-gitops` repo from [github](https://github.com/utkarsh-homelab/homelab-gitops)

### bootstrap/argocd-install.yaml

Generate the bootstrap YAML from the umbrella chart with our values:

```bash
cd ~/Projects/Homelab/homelab-infra-charts

# Add upstream Helm repo
helm repo add argo https://argoproj.github.io/argo-helm

# Download sub-chart dependencies so helm template can render them
helm dependency build charts/argocd/

# Generate the bootstrap YAML
helm template argocd charts/argocd/ \
  --namespace argocd --create-namespace \
  --skip-crds \
  --values charts/argocd/values/prod.yaml \
  > ../homelab-gitops/bootstrap/argocd-install.yaml
```

This generates raw Kubernetes manifests (Deployments, Services, etc.) for the initial install. CRDs are excluded here - they'll be installed from upstream URLs in Step 3 instead. The bootstrap file is committed to the `gitops` repo.

### apps/root.yaml

The App of Apps entry point:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/utkarsh-homelab/homelab-gitops
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
```

### apps/argocd.yaml (Self-Management)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
    targetRevision: HEAD
    path: charts/argocd
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> All other child Application manifests (`metallb.yaml`, `traefik.yaml`, etc. that we'll create in future) follow the same pattern - same `repoURL`, different `path`, and `valueFiles: [values/prod.yaml]`.

## Step 3: Install CRDs

The bootstrap YAML uses `--skip-crds` to avoid Kubernetes' 256KB annotation limit on the ApplicationSets CRD. Install CRDs from upstream first:

```bash
# Verify kubeconfig is configured properly
kubectl get nodes

# Install ArgoCD CRDs from upstream (no Helm annotations — avoids size limit)
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.4/manifests/crds/application-crd.yaml

kubectl create -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.4/manifests/crds/applicationset-crd.yaml

kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.4/manifests/crds/appproject-crd.yaml
```

## Step 4: Bootstrap ArgoCD

```bash
# Verify kubeconfig is configured properly
kubectl get nodes

# Apply the bootstrap YAML from GitHub
kubectl create namespace argocd

# Use --server-side to avoid the 256KB annotation limit
curl -sL https://raw.githubusercontent.com/utkarsh-homelab/homelab-gitops/main/bootstrap/argocd-install.yaml \
  | kubectl apply --server-side -n argocd -f -

# Wait for ArgoCD to be ready
kubectl wait --for=condition=ready pod -n argocd -l app.kubernetes.io/name=argocd-server --timeout=300s
```

## Step 5: Apply the Root App of Apps

```bash
kubectl apply -f https://raw.githubusercontent.com/utkarsh-homelab/homelab-gitops/main/apps/root.yaml
```

## Bootstrap Flow Summary

```
1. Manual: kubectl apply -f upstream CRDs (application-crds.yaml + applicationset-crds.yaml)

2. Manual: kubectl apply -f bootstrap/argocd-install.yaml

3. Manual: kubectl apply -f apps/root.yaml

4. ArgoCD syncs root → discovers apps/ directory

5. root creates Application resources for each component

6. Each Application syncs its respective umbrella chart from homelab-infra-charts

7. ArgoCD manages itself via apps/argocd.yaml → homelab-infra-charts/charts/argocd
```

## Bootstrap Access Note

At bootstrap time, MetalLB and Traefik do not exist yet (I'll create guide for them next), so the ArgoCD Ingress won't work. Use port-forwarding for initial access:

```bash
# Get login password, the default username is `admin`
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# Port forward argocd-server service
kubectl port-forward svc/argocd-server -n argocd 8080:443 
```

> The Ingress becomes active after MetalLB and Traefik are synced (sync waves -1 and 0 complete). I'll create a guide for deploying them later 😊.

## Sync Wave Strategy

| Wave | App | Purpose |
|------|-----|---------|
| -2 | argocd | Self-management (installed first) |
| -1 | metallb | Provides LoadBalancer IPs |
| 0 | traefik, cert-manager | Ingress + TLS |
| 1 | kube-prometheus-stack, csi-driver-nfs | Monitoring + storage |

> [!NOTE]
> I'll be creating guides for other components too. You can find them in this repo once they are created.

## References

- [ArgoCD Helm Chart](https://artifacthub.io/packages/helm/argo/argo-cd)
- [ArgoCD Self-Managed Patterns](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps)
