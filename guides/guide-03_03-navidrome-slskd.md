# 03-03: Deploy Navidrome + slskd (Soulseek) via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy.
- [ ] ArgoCD bootstrapped and running
- [ ] Root app synced and healthy
- [ ] apps-root synced and healthy
- [ ] MetalLB deployed
- [ ] Traefik + cert-manager deployed
- [ ] CSI Driver NFS deployed
- [ ] Shared media PVC `media-share` created in namespace `media`
- [ ] Pi-hole DNS configured for `*.uttutu.xyz` → `192.168.0.200`

## What We're Building

| Component | Version | Purpose |
|-----------|---------|---------|
| Navidrome | latest | Music streaming server (Subsonic-compatible) |
| slskd | latest | Soulseek client with web UI |
| MetalLB LB | — | Direct external access for Soulseek network port |

Both apps share the `music` folder on the `media-share` PVC. slskd downloads music there, Navidrome serves it.

## Shared Media Storage

### Update media subdirectories

Run the utility pod again to add the `music` folder:

```bash
kubectl apply -n media -f - <<'EOF'
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
          mkdir -p /media/music
          echo "Done. Directories:"
          ls -la /media
      volumeMounts:
        - name: media
          mountPath: /media
  volumes:
    - name: media
      persistentVolumeClaim:
        claimName: media-share
EOF

kubectl wait -n media --for=condition=ready pod/init-media-dirs --timeout=30s
kubectl logs -n media init-media-dirs
kubectl delete -n media pod/init-media-dirs
```

### Media Layout

```
media-share PVC:
├── books/            ← Calibre-Web-Automated
├── cwa-ingest/       ← Calibre-Web-Automated
└── music/            ← slskd downloads + Navidrome serves
```

## Navidrome Custom Chart

> [!NOTE]
> The chart is in my [homelab-app-charts repo](https://github.com/utkarsh-homelab/homelab-app-charts).
>
> `git clone git@github.com:utkarsh-homelab/homelab-app-charts.git`

### charts/navidrome/Chart.yaml

```yaml
apiVersion: v2
name: navidrome
description: Custom Helm chart for Navidrome for uttutu homelab
type: application
version: 1.0.0
appVersion: latest
```

### charts/navidrome/values/prod.yaml

```yaml
navidrome:
  replicaCount: 1
  strategyType: Recreate

  image:
    repository: deluan/navidrome
    tag: latest
    pullPolicy: Always

  env:
    ND_MUSICFOLDER: "/music"
    ND_DATAFOLDER: "/data"

  service:
    port: 4533

  persistence:
    config:
      enabled: true
      size: 1Gi
      storageClassName: nfs-csi
      accessMode: ReadWriteOnce
    music:
      enabled: true
      existingClaim: "media-share"
      subPath: "music"

  ingress:
    enabled: true
    ingressClassName: traefik
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - host: music.uttutu.xyz
        paths:
          - path: /
    tls:
      - secretName: music-tls
        hosts:
          - music.uttutu.xyz

  securityContext:
    runAsUser: 2000
    runAsGroup: 2000
    fsGroup: 2000

  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi
      cpu: 500m
```

> [!NOTE]
> - Navidrome runs as UID 2000 (must match the `PUID`/`PGID` set for slskd and CWA).
> - `/music` is mounted read-only from the shared PVC. Music files are managed by slskd.
> - The `/data` folder (config + DB) is on its own dedicated PVC with permission init.

## slskd (Soulseek) Custom Chart

### charts/slskd/Chart.yaml

```yaml
apiVersion: v2
name: slskd
description: Custom Helm chart for slskd (Soulseek) for uttutu homelab
type: application
version: 1.0.0
appVersion: latest
```

### charts/slskd/values/prod.yaml

```yaml
slskd:
  replicaCount: 1
  strategyType: Recreate

  image:
    repository: slskd/slskd
    tag: latest
    pullPolicy: Always

  env:
    PUID: "2000"
    PGID: "2000"
    SLSKD_REMOTE_CONFIGURATION: "true"
    SLSKD_SHARED_DIR: "/shared"
    SLSKD__SOULSEEK__PORT: "50300"
    SLSKD__DIRECTORIES__DOWNLOADS: "/shared"
    SLSKD__DIRECTORIES__INCOMPLETE: "/shared"

  service:
    port: 5030
    slskPort: 50300
    loadBalancerIP: "192.168.0.201"

  persistence:
    config:
      enabled: true
      size: 1Gi
      storageClassName: nfs-csi
      accessMode: ReadWriteOnce
    shared:
      enabled: true
      existingClaim: "media-share"
      subPath: "music"

  ingress:
    enabled: true
    ingressClassName: traefik
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - host: soulseek.uttutu.xyz
        paths:
          - path: /
    tls:
      - secretName: soulseek-tls
        hosts:
          - soulseek.uttutu.xyz

  resources:
    requests:
      memory: 512Mi
      cpu: 200m
    limits:
      memory: 1Gi
      cpu: 500m
```

> [!NOTE]
> - slskd creates two services: a ClusterIP (`slskd`) for the web UI behind Traefik, and a LoadBalancer (`slskd-slsk`) for the Soulseek listening port.
> - The LoadBalancer gets IP `192.168.0.201` from the MetalLB pool (`192.168.0.200-192.168.0.249`).
> - Soulseek listens on port `50300` — ensure this port is forwarded on your router for full connectivity.
> - `SLSKD_REMOTE_CONFIGURATION=true` allows configuring settings from the web UI.
> - `SLSKD_SHARED_DIR=/shared` — directory you share for uploads, which backs to the `music` folder on the shared PVC.
> - `SLSKD__DIRECTORIES__DOWNLOADS=/shared` — where completed downloads land (must match the shared volume mount).
> - `SLSKD__DIRECTORIES__INCOMPLETE=/shared` — where in-progress downloads are stored.

### Port Forwarding for Soulseek

For full Soulseek connectivity, forward the listening port on your router:

| Protocol | External Port | Destination |
|----------|--------------|-------------|
| TCP | 50300 | `192.168.0.201:50300` |

## GitOps Apps

> [!NOTE]
> The GitOps repo can be found [here](https://github.com/utkarsh-homelab/homelab-gitops)

Both Application manifests go in `self-hosted-apps/` (same as CWA) with sync wave 10:

### self-hosted-apps/navidrome.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: navidrome
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
    path: charts/navidrome
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

### self-hosted-apps/slskd.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: slskd
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
    path: charts/slskd
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

Add these to Pi-hole:

| Domain | IP |
|--------|-----|
| `music.uttutu.xyz` | `192.168.0.200` |
| `soulseek.uttutu.xyz` | `192.168.0.200` |

## Trigger ArgoCD Sync

```bash
argocd app sync root
```

Sync chain: `infra-root` → `apps-root` (wave 5) → `navidrome` + `slskd` (wave 10).

## Verify

```bash
kubectl get pods -n media
kubectl get pvc -n media
kubectl get ingress -n media
kubectl get svc -n media
```

Check the LoadBalancer got its IP:

```bash
kubectl get svc -n media slskd-slsk
```

Expect `EXTERNAL-IP` to show `192.168.0.201`.

## Access

### Navidrome

```bash
kubectl port-forward -n media svc/navidrome 4533:4533 &
```

Open `http://localhost:4533` — create your admin account on first visit.

Once DNS propagates: **https://music.uttutu.xyz**

### slskd

```bash
kubectl port-forward -n media svc/slskd 5030:5030 &
```

Open `http://localhost:5030` and log in with the **default credentials**:

| Field | Value |
|-------|-------|
| Username | `slskd` |
| Password | `slskd` |

> [!IMPORTANT]
> Change the default password immediately after first login.

Once DNS propagates: **https://soulseek.uttutu.xyz**

### Configure slskd

1. Log in to the web UI
2. Go to **Configuration** → **Soulseek**
3. Enter your Soulseek **username** and **password**
4. Configure your **Soulseek username** and **password**
5. Set **listening port** to `50300`
6. Save and restart

The download directories are already configured via `SLSKD__DIRECTORIES__DOWNLOADS=/shared` and `SLSKD__DIRECTORIES__INCOMPLETE=/shared` in the chart env vars, so files land on the shared music PVC.

## Summary

| Component | Namespace | Ingress URL | LB IP |
|-----------|-----------|-------------|-------|
| Navidrome | media | https://music.uttutu.xyz | — |
| slskd (Web UI) | media | https://soulseek.uttutu.xyz | — |
| slskd (Soulseek) | media | — | `192.168.0.201:50300` |

## References

- [Navidrome Docker](https://www.navidrome.org/docs/installation/docker/)
- [Navidrome Configuration](https://www.navidrome.org/docs/usage/configuration/options/)
- [slskd GitHub](https://github.com/slskd/slskd)
- [slskd Docker Docs](https://github.com/slskd/slskd/blob/master/docs/docker.md)
- [slskd Configuration](https://github.com/slskd/slskd/blob/master/docs/config.md)
