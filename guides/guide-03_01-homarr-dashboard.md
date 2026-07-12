# 03-01: Deploy Homarr via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy.
- [ ] ArgoCD bootstrapped and running (Guide 2)
- [ ] Root app synced and healthy
- [ ] MetalLB deployed
- [ ] Traefik + cert-manager deployed
- [ ] CSI Driver NFS deployed
- [ ] Pi-hole DNS configured for `*.uttutu.xyz` → `192.168.0.200`

## What We're Building

Homarr - a sleek, modern dashboard for your homelab:

| Component | Upstream Version | Purpose |
|-----------|-----------------|---------|
| Homarr | v1.70.0 | Self-hosted dashboard with service widgets |
| Kubernetes RBAC | - | Cluster integration for resource monitoring |
| NFS-backed PVC | - | Persistent database storage |

## Homarr Umbrella Chart

> [!NOTE]
> I have already created the chart in my [homelab-app-charts repo](https://github.com/utkarsh-homelab/homelab-app-charts), So you just need to clone the repo 😇. Change the values according to your cluster setup.
>
> `git clone git@github.com:utkarsh-homelab/homelab-app-charts.git`

### charts/homarr/Chart.yaml

```yaml
apiVersion: v2
name: homarr
description: Vendored Homarr Helm chart for uttutu homelab
type: application
version: 1.0.0
appVersion: v1.70.0
dependencies:
  - name: homarr
    version: 8.22.0
    repository: https://homarr-labs.github.io/charts/
```

### charts/homarr/values/prod.yaml

```yaml
homarr:
  env:
    TZ: "Asia/Kolkata"

  strategyType: Recreate

  persistence:
    homarrDatabase:
      enabled: true
      storageClassName: nfs-csi
      accessMode: ReadWriteOnce
      size: 1Gi

  ingress:
    enabled: true
    ingressClassName: traefik
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - host: homarr.uttutu.xyz
        paths:
          - path: /
    tls:
      - secretName: homarr-tls
        hosts:
          - homarr.uttutu.xyz

  rbac:
    enabled: true
```

### Create the encryption secret

The upstream chart requires a `SECRET_ENCRYPTION_KEY` for encrypting session data and database fields. Create it as a Kubernetes Secret before deploying - this keeps the key out of your values file:

```bash
# Create the namespace first
kubectl create namespace homarr

# Generate and create the encryption secret
kubectl create secret generic db-encryption \
  --namespace homarr \
  --from-literal=db-encryption-key="$(openssl rand -hex 32)"
```

The upstream chart defaults to looking for a secret named `db-encryption` with key `db-encryption-key`, so no additional configuration in values is needed.

### Download sub-chart dependencies

```bash
cd ./homelab-app-charts

# Add upstream Helm repo
helm repo add homarr-labs https://homarr-labs.github.io/charts/

# Download sub-chart dependencies
helm dependency build charts/homarr/
```

## GitOps App Structure

> [!NOTE]
> The GitOps repo can be found [here](https://github.com/utkarsh-homelab/homelab-gitops)

We need two new files in the gitops repo:

### 1. apps-root Application

The `infra-apps/apps-root.yaml` acts as the App-of-Apps entry point for self-hosted applications. It references the `apps/` directory and deploys on sync wave 5 (after all infra components at waves -2..1 are ready):

```yaml
# gitops/infra-apps/apps-root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-root
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "5"
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

### 2. Homarr Application

The `apps/homarr.yaml` references the app-charts repo on sync wave 10:

```yaml
# gitops/apps/homarr.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: homarr
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "10"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/utkarsh-homelab/homelab-app-charts
    targetRevision: HEAD
    path: charts/homarr
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: homarr
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Sync Wave Summary

| Wave | App | Purpose |
|------|-----|---------|
| -2 | argocd | Self-management |
| -1 | metallb | LoadBalancer IPs |
| 0 | traefik, cert-manager | Ingress + TLS |
| 1 | csi-driver-nfs, kube-prometheus-stack | Storage + monitoring |
| 5 | apps-root | App-of-apps entry point |
| 10 | homarr | Dashboard |

## Verify Pi-hole DNS Records

No Cloudflare DNS records needed. Add this to Pi-hole:

| Domain | IP |
|--------|-----|
| `homarr.uttutu.xyz` | `192.168.0.200` |

## Trigger ArgoCD Sync

```bash
argocd app sync root
```

This will sync `infra-root`, which discovers and syncs `apps-root` at wave 5, which in turn discovers and syncs `homarr` at wave 10.

## Verify

```bash
kubectl get pods -n homarr
kubectl get pvc -n homarr
kubectl get ingress -n homarr
kubectl get serviceaccount -n homarr
kubectl get clusterrole -l app.kubernetes.io/instance=homarr
```

## Access Homarr

```bash
kubectl port-forward -n homarr svc/homarr 7575:7575 &
```

Open `http://localhost:7575` - configure your dashboard, add integrations, and set up authentication.

Once DNS propagates, the dashboard will be available at **https://homarr.uttutu.xyz**

## CoreDNS Forwarding for Local DNS

Homarr pods use CoreDNS (cluster DNS), not Pi-hole directly. (This might cause issues when using the ping feature in the app widgets since it can't resolve the DNS.)
If `*.uttutu.xyz` domains don't resolve from inside pods, add a forward zone in CoreDNS:

```bash
kubectl edit configmap coredns -n kube-system
```
Append a uttutu.xyz server block:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          prefer_udp
        }
        cache 30
        loop
        reload
        loadbalance
    }
    uttutu.xyz:53 {
        errors
        cache 30
        forward . 192.168.0.101
    }
```

This forwards all `*.uttutu.xyz` queries to Pi-hole at `192.168.0.101`, which returns `192.168.0.200` (Traefik's LoadBalancer IP).

Verify DNS Resolution from Pods

- Test CoreDNS → Pi-hole chain
```bash
kubectl run tmp-dns --image=busybox -it --rm --restart=Never -- nslookup homarr.uttutu.xyz
```

- Test Pi-hole directly (bypass CoreDNS)
```bash
kubectl run tmp-dns --image=busybox -it --rm --restart=Never -- nslookup homarr.uttutu.xyz 192.168.0.101
```

If the first fails but the second succeeds, CoreDNS forwarding isn't configured yet.

## Summary

| Component | Namespace | Ingress URL |
|-----------|-----------|-------------|
| Homarr | homarr | https://homarr.uttutu.xyz |

## What's Next

The Kubernetes integration is enabled via `rbac.enabled: true`, which creates a ServiceAccount, Role, ClusterRole, and their bindings. Inside Homarr, you can add a **Kubernetes** widget to monitor pods, deployments, and other resources in real-time from the dashboard.

## References

- [Homarr Helm Chart](https://artifacthub.io/packages/helm/homarr-labs/homarr)
- [Homarr Docs](https://homarr.dev/docs/getting-started/installation/helm/)
- [Homarr Charts Source](https://github.com/homarr-labs/charts)
