# 02-07: Deploy MetalLB via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy.
- [ ] ArgoCD bootstrapped and running (Guide 2)
- [ ] Root app synced and healthy
- [ ] Router configured to keep IP pool (192.168.0.200-192.168.0.249) out of DHCP range

## What We're Building

MetalLB in Layer2 mode with an IP pool of `192.168.0.200-192.168.0.249`. This provides LoadBalancer IPs for services.

> [!TIP]
> I have configured my router to keep this IP pool out of the DHCP range you will have to do the same for the IP pool you want to use.

## MetalLB Umbrella Chart

> [!NOTE]
> I have already created the chart in my [homelab-infra-charts repo](https://github.com/utkarsh-homelab/homelab-infra-charts), So you just need to clone the repo 😇
>
> `git clone git@github.com:utkarsh-homelab/homelab-infra-charts.git`

### charts/metallb/Chart.yaml

```yaml
apiVersion: v2
name: metallb
description: Vendored MetalLB Helm chart for uttutu homelab
type: application
version: 1.0.0
appVersion: v0.14.9
dependencies:
  - name: metallb
    version: 0.16.1
    repository: https://metallb.github.io/metallb
```

### charts/metallb/values/prod.yaml

```yaml
metallb:
  frrk8s:
    enabled: false
  crds:
    enabled: true
    validationFailurePolicy: Fail
```

### charts/metallb/templates/ip-pool.yaml

MetalLB v0.14+ uses CRDs for configuration.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: {{ .Release.Namespace }}
spec:
  addresses:
    - 192.168.0.200-192.168.0.249
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: {{ .Release.Namespace }}
spec:
  ipAddressPools:
    - default-pool
```

### Download sub-chart dependencies

```bash
cd ./homelab-infra-charts

# Add upstream Helm repo
helm repo add metallb https://metallb.github.io/metallb

# Download sub-chart dependencies
helm dependency build charts/metallb/
```

## MetalLB GitOps App

> [!NOTE]
> The GitOps repo can be found [here](https://github.com/utkarsh-homelab/homelab-gitops)

The `apps/metallb.yaml` references the infra-charts repo on sync wave -1:

> refer to the [gitops repo](https://github.com/utkarsh-homelab/homelab-gitops) for more details on sync strategy.

```yaml
homelab-gitops/apps/metallb.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
    targetRevision: HEAD
    path: charts/metallb
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: metallb-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Trigger ArgoCD Sync

```bash
argocd app sync root
```

## Verify MetalLB

```bash
# commands with sample outputs

kubectl get pods -n metallb-system
▶ NAME                                              READY   STATUS    RESTARTS   AGE
▶ metallb-controller-7b8f6f8f6f-abcde               1/1     Running   0          1m
▶ metallb-speaker-5b87dfd986-qxc7k                  1/1     Running   0          1m

kubectl get ipaddresspools -n metallb-system
▶ NAME           AGE
▶ default-pool   1m

kubectl get l2advertisements -n metallb-system
▶ NAME          AGE
▶ default-l2    1m
```

MetalLB is providing LoadBalancer IPs.

## References

- [MetalLB Helm Chart](https://artifacthub.io/packages/helm/metallb/metallb)
- [MetalLB Layer2 Configuration](https://metallb.universe.tf/configuration/)
- [MetalLB Github](https://github.com/metallb/metallb)