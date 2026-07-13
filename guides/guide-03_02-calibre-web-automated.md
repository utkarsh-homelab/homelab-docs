# 03-02: Deploy Calibre-Web-Automated via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy.
- [ ] ArgoCD bootstrapped and running (Guide 2)
- [ ] Root app synced and healthy
- [ ] apps-root synced and healthy
- [ ] MetalLB deployed
- [ ] Traefik + cert-manager deployed
- [ ] CSI Driver NFS deployed
- [ ] Shared media PVC created (see [Shared Media Storage](#shared-media-storage) below)
- [ ] Pi-hole DNS configured for `*.uttutu.xyz` → `192.168.0.200`

## What We're Building

Calibre-Web Automated (CWA) — all-in-one eBook management with web UI, metadata fetching, and automated import:

| Component | Version | Purpose |
|-----------|---------|---------|
| Calibre-Web Automated | latest | eBook library web interface + automations |
| Config PVC | — | Dedicated app config (cwa-config) |
| Shared Media PVC | — | Unified media volume shared across apps |

## Shared Media Storage

All media apps mount a single `ReadWriteMany` PVC backed by NFS-CSI. Each app accesses its own subdirectory via `subPath`. This keeps files accessible to both downloaders (qBittorrent, Soulseek) and servers (Jellyfin, Navidrome, CWA).

### Create media namespace

```bash
kubectl create ns media
```

### Create the shared PVC

```yaml
# media-share-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: media-share
  namespace: media
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 100Gi
```

```bash
kubectl apply -f media-share-pvc.yaml
```

### Create media subdirectories

Create a utility pod that mounts the PVC and creates the CWA directories:

```yaml
# media-init-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-media-dirs
spec:
  restartPolicy: Never
  containers:
    - name: init-dirs
      image: busybox
      command:
        - /bin/sh
        - -c
        - |
          mkdir -p /media/books /media/cwa-ingest
          echo "Done. Directories:"
          ls -la /media
      volumeMounts:
        - name: media
          mountPath: /media
  volumes:
    - name: media
      persistentVolumeClaim:
        claimName: media-share
```

```bash
kubectl apply -n media -f media-init-pod.yaml
```

Wait for it to complete and check the output:

```bash
kubectl wait -n media --for=condition=ready pod/init-media-dirs --timeout=30s
kubectl logs -n media init-media-dirs
```

Remove the utility pod:

```bash
kubectl delete -n media pod/init-media-dirs
```

> [!NOTE]
> When we'll deploy future apps (qBittorrent, Soulseek, Jellyfin, Navidrome), we'll run this utility pod again to create the additional subdirectories and symlinks on the same `media-share` PVC.

## Calibre-Web-Automated Custom Chart

> [!NOTE]
> The chart is in my [homelab-app-charts repo](https://github.com/utkarsh-homelab/homelab-app-charts).
>
> `git clone git@github.com:utkarsh-homelab/homelab-app-charts.git`

Since there is no official Helm chart for CWA, I created a custom chart with templates derived from the official Kubernetes manifests. It supports both dedicated PVCs (standalone mode) and shared PVCs with `subPath` (media-homelab mode).

### charts/calibre-web-automated/Chart.yaml

```yaml
apiVersion: v2
name: calibre-web-automated
description: Custom Helm chart for Calibre-Web-Automated for uttutu homelab
type: application
version: 1.0.0
appVersion: latest
```

### charts/calibre-web-automated/values/prod.yaml

```yaml
calibre_web_automated:
  replicaCount: 1
  strategyType: Recreate

  image:
    repository: crocodilestick/calibre-web-automated
    tag: latest
    pullPolicy: Always

  env:
    TZ: "Asia/Kolkata"
    PUID: "2000"
    PGID: "2000"
    NETWORK_SHARE_MODE: "true"
    TRUSTED_PROXY_COUNT: "1"
    CWA_PORT_OVERRIDE: "8083"

  service:
    port: 8083

  persistence:
    config:
      enabled: true
      size: 5Gi
      storageClassName: nfs-csi
      accessMode: ReadWriteOnce
    library:
      enabled: true
      existingClaim: "media-share"
      subPath: "books"
      size: 50Gi
      storageClassName: nfs-csi
      accessMode: ReadWriteOnce
    ingest:
      enabled: true
      existingClaim: "media-share"
      subPath: "cwa-ingest"
      size: 10Gi
      storageClassName: nfs-csi
      accessMode: ReadWriteOnce

  ingress:
    enabled: true
    ingressClassName: traefik
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - host: books.uttutu.xyz
        paths:
          - path: /
    tls:
      - secretName: books-tls
        hosts:
          - books.uttutu.xyz

  resources:
    requests:
      memory: 512Mi
      cpu: 100m
    limits:
      memory: 1Gi
      cpu: 500m
```

> [!NOTE]
> - `library.existingClaim: media-share` — Mounts the shared PVC instead of creating a dedicated one.
> - `library.subPath: books` — Mounts only the `books/` subdirectory inside the shared PVC.
> - `ingest.existingClaim: media-share` + `subPath: cwa-ingest` — Same shared PVC, different subdirectory.
> - When `existingClaim` is empty, the chart creates dedicated PVCs (`calibre-library-pvc`, `cwa-ingest`).
> - `NETWORK_SHARE_MODE=true` — Required when library is on NFS. Disables WAL mode and switches file watching to polling for reliability.
> - `TRUSTED_PROXY_COUNT=1` — Traefik is the only reverse proxy in front of CWA.

### Download sub-chart dependencies

This is a self-contained chart with no external dependencies, so no `helm dependency build` is needed.

## Storage Layout

| Volume | Source | Size | Purpose |
|--------|--------|------|---------|
| `/config` | PVC `cwa-config` | 5Gi | Application configuration and state |
| `/calibre-library` | PVC `media-share` subPath `books` | — | Calibre eBook library |
| `/cwa-book-ingest` | PVC `media-share` subPath `cwa-ingest` | — | Auto-import folder |

> [!TIP]
> The ingest folder is where you drop eBook files for automatic import. Once CWA processes them, they are added to your library and the originals are deleted.

## GitOps App

> [!NOTE]
> The GitOps repo can be found [here](https://github.com/utkarsh-homelab/homelab-gitops)

The `self-hosted-apps/calibre-web-automated.yaml` Application manifest references the app-charts repo:

```yaml
# gitops/self-hosted-apps/calibre-web-automated.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: calibre-web-automated
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
    path: charts/calibre-web-automated
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: media
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

## Verify Pi-hole DNS Records

Add this to Pi-hole:

| Domain | IP |
|--------|-----|
| `books.uttutu.xyz` | `192.168.0.200` |

## Trigger ArgoCD Sync

```bash
argocd app sync root
```

The sync chain will be: `infra-root` → `apps-root` (wave 5) → `calibre-web-automated` (wave 10).

## Verify

```bash
kubectl get pods -n media
kubectl get pvc -n media
kubectl get ingress -n media
```

Wait for the pod to become ready — the initial startup can take a couple of minutes as CWA sets up the Calibre environment and metadata database:

```bash
kubectl wait --for=condition=ready pod -n media -l app.service=calibre-web-automated --timeout=300s
```

Check that the shared PVC is mounted correctly:

```bash
kubectl exec -n media deploy/calibre-web-automated -- ls -la /calibre-library
kubectl exec -n media deploy/calibre-web-automated -- ls -la /cwa-book-ingest
```

## Access Calibre-Web-Automated

```bash
kubectl port-forward -n media svc/calibre-web-automated 8083:8083 &
```

Open `http://localhost:8083` and log in with the **default admin credentials**:

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `admin123` |

> [!IMPORTANT]
> Change the default password immediately after first login.

Once DNS propagates, the full URL will be **https://books.uttutu.xyz**

## Post-Install Tasks

After first login, navigate to **Settings → Basic Configuration** and enable at least:

- [ ] **Enable Uploads** — Required to upload books from the web UI
- [ ] Configure your library path: `/calibre-library`

## Summary

| Component | Namespace | Ingress URL |
|-----------|-----------|-------------|
| Calibre-Web-Automated | media | https://books.uttutu.xyz |
| Shared Media PVC | media | `media-share` (500Gi RWX) |

## References

- [CWA GitHub](https://github.com/crocodilestick/Calibre-Web-Automated)
- [CWA Wiki - Configuration](https://github.com/crocodilestick/Calibre-Web-Automated/wiki/Configuration)
- [CWA Docker Compose Reference](https://github.com/crocodilestick/Calibre-Web-Automated/blob/main/docker-compose.yml)
