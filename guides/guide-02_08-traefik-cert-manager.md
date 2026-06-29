# 02-07: Deploy MetalLB via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy.
- [ ] ArgoCD bootstrapped and running 
- [ ] Root app synced and healthy
- [ ] MetalLB deployed and working
- [ ] Cloudflare API token ready (we create it here)
- [ ] Pihole running (for local DNS records)

## What We're Building

| Component | Purpose | Version |
|-----------|---------|---------|
| Traefik | Ingress controller — handles all `*.uttutu.xyz` traffic | v3.7.5 |
| cert-manager | TLS certificates via Let's Encrypt DNS01 with Cloudflare | v1.17.0 |

## Part A: Create the Cloudflare API Token Secret (Manual)

To use the Cloudflare API for an ACME DNS-01 challenge, we need to create a securely scoped API Token with specific DNS permissions.

### Step-by-Step Token Creation

- Log in to your Cloudflare Dashboard.
- Click the User Profile icon in the top-right corner and select `Manage Account`.
- Choose the `Account API Tokens` tab from the left sidebar.
- Click the blue `Create a Token` button.

### Required Token Settings

Configure the token with these precise permissions to follow the principle of least privilege:

- Token Name: Give it a clear name (e.g., ACME DNS Challenge).
- Select your domain
- DNS & Zone Permissions:
    - Zone ➡️ DNS ➡️ Edit
    - Zone ➡️ Zone ➡️ Read (Note: This is required by some ACME clients like cert-manager to locate the correct zone ID).
- TTL / Expiry (Optional): Set to `No expiration` for continuous automatic renewals, or set an expiration date if it is for a one-time test.

Click Continue to summary, verify the permissions, and click Create Token.

> [!WARNING]
> Copy your API token secret immediately and save it in a secure password manager. Cloudflare will only show this token value once.


```bash
# Create the namespace
kubectl create namespace cert-manager

# Create secret with your Cloudflare API token
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token=<your-token-from-cloudflare>

# Verify
kubectl get secret cloudflare-api-token -n cert-manager
```

> This is the only manual step. The secret is not managed by ArgoCD. I'll probably have it managed by ArgoCD in the future after I have deployed and configured ExternalSecrets in the clutser.

### DNS Flow: Private A Records, Public TXT Records

cert-manager uses Cloudflare DNS01 challenge to issue the wildcard certificate. This works by creating **TXT records** at Cloudflare (public) for Let's Encrypt to validate. The **A records** that consumers use (`argocd.uttutu.xyz` → `192.168.0.200`) are created only in Pi-hole and never reach Cloudflare.

```
cert-manager → Cloudflare API → creates _acme-challenge.uttutu.xyz TXT → Let's Encrypt validates
                                                                                ↓
Laptop → Pi-hole → argocd.uttutu.xyz A → 192.168.0.200 → Traefik → HTTPS (wildcard cert)
                     (local only, no Cloudflare A record)
```

This means:
- **Cloudflare**: Only TXT records for the DNS01 challenge. No A records → services are unreachable from the internet.
- **Pi-hole**: All A records for `*.uttutu.xyz` → `192.168.0.200`. Only resolvable on the local network.
- **Wildcard TLS certificate**: Real Let's Encrypt cert, valid and trusted everywhere.

> [!NOTE]
> This is the setup for now, in future when I want to expose services to the internet, I'll add the details in the cloudlflare tunnel configurations so traffic from the internet can reach traefik in the cluster but only for the routes that we want. 
>
> For now everything is private and only accessible from the LAN.

## Part B: cert-manager Umbrella Chart

> [!NOTE]
> I have already created the chart in my [homelab-infra-charts repo](https://github.com/utkarsh-homelab/homelab-infra-charts), So you just need to clone the repo 😇
>
> `git clone git@github.com:utkarsh-homelab/homelab-infra-charts.git`

### charts/cert-manager/Chart.yaml

```yaml
apiVersion: v2
name: cert-manager
description: Vendored cert-manager Helm chart for uttutu homelab
type: application
version: 1.0.0
appVersion: v1.17.0
dependencies:
  - name: cert-manager
    version: 1.17.0
    repository: https://charts.jetstack.io
```

### charts/cert-manager/values/prod.yaml

```yaml
cert-manager:
  installCRDs: true
  replicaCount: 1
  dns01RecursiveNameservers: "192.168.0.101:53"
  dns01RecursiveNameserversOnly: true

  config:
    apiVersion: controller.config.cert-manager.io/v1alpha1
    kind: ControllerConfiguration
    logging:
      verbosity: 2
      format: text

  webhook:
    replicaCount: 1

  cainjector:
    replicaCount: 1

  prometheus:
    servicemonitor:
      enabled: true
      interval: 30s
```

### charts/cert-manager/manifests/cluster-issuer.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: utkarsh.singh27800@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

### charts/cert-manager/manifests/wildcard-certificate.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-uttutu
  namespace: cert-manager
spec:
  secretName: wildcard-uttutu-tls
  commonName: "*.uttutu.xyz"
  dnsNames:
    - "*.uttutu.xyz"
    - "uttutu.xyz"
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  renewBefore: 360h
```

### Download sub-chart dependencies

```bash
cd ./homelab-infra-charts

# Add upstream Helm repo
helm repo add cert-manager https://charts.jetstack.io

# Download sub-chart dependencies
helm dependency build charts/cert-manager
```

## Part C: Traefik Umbrella Chart

> [!NOTE]
> Again, I have already created the chart in my [homelab-infra-charts repo](https://github.com/utkarsh-homelab/homelab-infra-charts), So you just need to clone the repo 😇
>
> `git clone git@github.com:utkarsh-homelab/homelab-infra-charts.git`

### charts/traefik/Chart.yaml

```yaml
apiVersion: v2
name: traefik
description: Vendored Traefik Helm chart for uttutu homelab
type: application
version: 1.0.0
appVersion: v3.7.5
dependencies:
  - name: traefik
    version: 41.0.0
    repository: https://traefik.github.io/charts
```

### charts/traefik/values/prod.yaml

```yaml
traefik:
  deployment:
    kind: Deployment
    replicas: 1

  service:
    type: LoadBalancer
    spec:
      loadBalancerIP: 192.168.0.200
      externalTrafficPolicy: Local

  ingressClass:
    enabled: true
    isDefaultClass: true

  ingressRoute:
    dashboard:
      enabled: true
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      matchRule: Host(`traefik.uttutu.xyz`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
      entryPoints:
        - websecure

  ports:
    web:
      port: 8000
      expose:
        default: true
      exposedPort: 80
      http:
        redirections:
          entryPoint:
            to: websecure
            scheme: https
            permanent: true
    websecure:
      port: 8443
      expose:
        default: true
      exposedPort: 443
    metrics:
      port: 9100
      expose:
        default: true
      exposedPort: 9100

  providers:
    kubernetesCRD:
      enabled: true
      allowCrossNamespace: true
      allowExternalNameServices: true
    kubernetesIngress:
      enabled: true
      publishedService:
        enabled: true

  tlsStore:
    default:
      defaultCertificate:
        secretName: wildcard-uttutu-tls

  metrics:
    prometheus:
      entryPoint: metrics
      addEntryPointsLabels: true
      addServicesLabels: true
      service:
        enabled: true
      serviceMonitor:
        enabled: true
        interval: 30s

  log:
    level: INFO

  accessLog:
    enabled: true

  tlsOptions:
    default:
      minVersion: VersionTLS12
      cipherSuites:
        - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
        - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256

  experimental:
    plugins: {}

  global:
    checkNewVersion: false
    sendAnonymousUsage: false
```

### Download sub-chart dependencies

```bash
cd ./homelab-infra-charts

# Add upstream Helm repo
helm repo add traefik https://traefik.github.io/charts

# Download sub-chart dependencies
helm dependency build charts/traefik/
```

## GitOps Repo Applications

> [!NOTE]
> The GitOps repo can be found [here](https://github.com/utkarsh-homelab/homelab-gitops)

> refer to the [gitops repo](https://github.com/utkarsh-homelab/homelab-gitops) for more details on sync strategy.

### cert-manager GitOps App

`apps/cert-manager.yaml` uses multi-source:

```yaml
# gitops/apps/cert-manager.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
      targetRevision: HEAD
      path: charts/cert-manager
      helm:
        valueFiles:
          - values/prod.yaml
    - repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
      targetRevision: HEAD
      path: charts/cert-manager/manifests
      directory:
        recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### traefik GitOps App

```yaml
# gitops/apps/traefik.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: traefik
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
    targetRevision: HEAD
    path: charts/traefik
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: traefik
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Part E: Trigger Sync

```bash
argocd app sync root
```

> ![WARNING]
> Cert-Manager and Traefik both have ServiceMonitor in their helm-charts but since the CRD for ServiceMonitor doesn't exist in the cluster yet the sync will **fail**, the sync will succeed once we have deployed the kube-prometheus-stack, which we'll be doing in the [next guide](./guide-02_09-kube-prometheus-stack.md).

## Part F: Verify

```bash
# cert-manager
kubectl get pods -n cert-manager
kubectl get clusterissuer
kubectl get certificate -n cert-manager
kubectl get secret -n cert-manager # check for wildcard-uttutu-*

# Traefik
kubectl get pods -n traefik
kubectl get svc -n traefik  # External-IP should be 192.168.0.200
kubectl get ingressclass    # traefik should be default
```

## Pi-hole Local DNS Records

Add these to Pi-hole's Local DNS → DNS Records (Admin UI at `http://192.168.0.101/admin`):

| Domain | IP |
|--------|-----|
| `argocd.uttutu.xyz` | `192.168.0.200` |
| `traefik.uttutu.xyz` | `192.168.0.200` |
| `grafana.uttutu.xyz` | `192.168.0.200` |
| `prometheus.uttutu.xyz` | `192.168.0.200` |
| `alertmanager.uttutu.xyz` | `192.168.0.200` |

> [!NOTE]
> No Cloudflare A records are needed for any of these. cert-manager's DNS01 challenge only needs Cloudflare API access (TXT records), not A records in Cloudflare DNS.
>
> If you want to expose service to the internet add the application route in the cloudflare tunnel configuration.

> [!NOTE]
> I'm adding the DNS record for prometheus, alertmanager and grafana here but we'll be deploying them in the [next guide](./guide-02_09-kube-prometheus-stack.md).

## References

- [Traefik Github](https://github.com/traefik/traefik)
- [Traefik Helm Chart](https://artifacthub.io/packages/helm/traefik/traefik)
- [Traefik Helm Chart Github](https://github.com/traefik/traefik-helm-chart)
- [cert-manager Helm Chart](https://artifacthub.io/packages/helm/cert-manager/cert-manager)
- [cert-manager Cloudflare DNS01](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/)
