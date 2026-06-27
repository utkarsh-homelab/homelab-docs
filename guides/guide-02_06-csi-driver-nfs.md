# 02-06: Deploy CSI-Driver-NFS via ArgoCD

## Prerequisites

- [ ] NFS share created on Proxmox host [Guide](./guide-01_06-proxmox-nfs-and-firewall.md)
- [ ] ArgoCD running and healthy [Guide - Manual](./guide-02_04-argocd-bootstrap-and-gitops-setup-manual.md) or [Guide - Automated](./guide-02_05-automating-argocd-bootstrap.md)

## What We're Building

| Component | Value |
|-----------|-------|
| NFS Server | 192.168.0.100 (Proxmox host) |
| NFS Export | `/var/nfs/kubernetes` |
| CSI Driver | csi-driver-nfs v4.13.3 |
| StorageClass | `nfs-csi` |
| Reclaim Policy | Retain |

## The CSI Driver NFS Umbrella Chart

> [!NOTE]
> I have already created the chart in my [homelab-infra-charts repo](https://github.com/utkarsh-homelab/homelab-infra-charts), So you just need to clone the repo 😇
>
> `git clone git@github.com:utkarsh-homelab/homelab-infra-charts.git`

### charts/csi-driver-nfs/Chart.yaml

```yaml
apiVersion: v2
name: csi-driver-nfs
description: Vendored CSI Driver NFS Helm chart for uttutu homelab
type: application
version: 1.0.0
appVersion: 4.13.3
dependencies:
  - name: csi-driver-nfs
    version: 4.13.3
    repository: https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
```

### charts/csi-driver-nfs/values/prod.yaml

Values are namespaced under `csi-driver-nfs:`. No image overrides — upstream defaults (`registry.k8s.io/sig-storage/*`) are used. The `storageClass` section configures the NFS server details.

> [!CAUTION]
> Update the `storageClass` section with your NFS server details.
>
> if you want help setting up the nfs share I have created a [Guide](./guide-01_06-proxmox-nfs-and-firewall.md) for Proxmox.

> [!CAUTION]
> I'm using `reclaimPolicy: Delete` since my cluster is a test cluster, change it according to your needs.

```yaml
csi-driver-nfs:
  driver:
    name: nfs.csi.k8s.io
    mountPermissions: 0

  feature:
    enableFSGroupPolicy: true
    enableInlineVolume: false

  kubeletDir: /var/lib/kubelet

  controller:
    name: csi-nfs-controller
    replicas: 1
    runOnControlPlane: false
    logLevel: 5
    workingMountDir: /tmp
    dnsPolicy: ClusterFirstWithHostNet

  node:
    name: csi-nfs-node
    logLevel: 5
    dnsPolicy: ClusterFirstWithHostNet
    maxUnavailable: 1

  externalSnapshotter:
    enabled: false

  storageClass:
    create: true
    name: nfs-csi
    parameters:
      server: 192.168.0.100
      share: /var/nfs/kubernetes
      subDir: "${pvc.metadata.namespace}/${pvc.metadata.name}"
      mountPermissions: "0755"
    reclaimPolicy: Delete 
    volumeBindingMode: Immediate
    mountOptions:
      - nfsvers=4.1
      - hard          # Explicitly enforce hard mounts
      - timeo=600     # Wait 60 seconds before retrying a request
      - retrans=2     # Retry twice before logging a warning
```

### Download sub-chart dependencies

```bash
cd ./homelab-infra-charts

# Add upstream Helm repo
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts

# Download sub-chart dependencies
helm dependency build charts/csi-driver-nfs/
```

## CSI Driver NFS GitOps App

> [!NOTE]
> The GitOps repo can be found [here](https://github.com/utkarsh-homelab/homelab-gitops)

The `apps/csi-driver-nfs.yaml` references the infra-charts repo on sync wave 1:

> refer to the [gitops repo](https://github.com/utkarsh-homelab/homelab-gitops) for more details on sync strategy.

```yaml
# homelab-gitops/apps/csi-driver-nfs.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: csi-driver-nfs
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
    targetRevision: HEAD
    path: charts/csi-driver-nfs
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: csi-nfs
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Trigger Sync

```bash
argocd app sync root
```

## Verify

```bash
kubectl get pods -n csi-nfs
kubectl get sc
```

## Test with a PVC

```yaml
# test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs-pod
spec:
  containers:
    - name: app
      image: nginx:latest
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-nfs-pvc
```

```bash
# Create pod and pvc
kubectl apply -f test-pvc.yaml

# Write a file to the volume
kubectl exec test-nfs-pod -- sh -c 'echo "Hello from NFS!" > /data/hello.txt'

# Read the file from the volume
kubectl exec test-nfs-pod -- cat /data/hello.txt

# Verify file on nfs-share on Proxmox
ssh root@<proxmox-ip> "cat /var/nfs/kubernetes/default/test-nfs-pvc/hello.txt"

# Cleanup (should delete the file from proxmox nfs-share)
kubectl delete pod test-nfs-pod
kubectl delete pvc test-nfs-pvc
```

## References

- [CSI Driver NFS](https://github.com/kubernetes-csi/csi-driver-nfs)
- [CSI NFS Helm Chart](https://artifacthub.io/packages/helm/csi-driver-nfs/csi-driver-nfs)