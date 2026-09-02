# 02-11: Deploy CloudNativePG PostgreSQL via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy
- [ ] ArgoCD bootstrapped and running [Guide - Manual](./guide-02_04-argocd-bootstrap-and-gitops-setup-manual.md) or [Guide - Automated](./guide-02_05-automating-argocd-bootstrap.md)
- [ ] Root app synced and healthy
- [ ] MetalLB deployed [Guide](./guide-02_07-metallb.md)
- [ ] Local-Path provisioner deployed [Guide](./guide-02_10-strimzi-kafka.md) (Part 1)

## What We're Building

| Component | Value |
|-----------|-------|
| CNPG Operator | v1.30.0 (Helm chart v0.29.0) |
| PostgreSQL Version | 16 |
| Instances | 1 (single primary) |
| Storage | Local-path provisioner (5Gi) |
| External Access | LoadBalancer on port 5432 via MetalLB |
| Namespace | `postgres` |
| Superuser Access | Disabled (use `app` user or `kubectl exec` for admin) |

> [!NOTE]
> CloudNativePG manages PostgreSQL workloads on Kubernetes. Unlike Strimzi, CNPG's Helm chart bundles CRDs — no separate manifest download needed.

---

## Part 1: CloudNativePG Operator

The CNPG operator watches for `Cluster` CRs and provisions PostgreSQL instances with streaming replication, backup, and failover support.

### charts/cloudnative-pg/Chart.yaml

```yaml
apiVersion: v2
name: cloudnative-pg
description: CloudNativePG Operator for PostgreSQL on Kubernetes
type: application
version: 1.0.0
appVersion: 1.30.0
dependencies:
  - name: cloudnative-pg
    version: 0.29.0
    repository: https://cloudnative-pg.github.io/charts
```

### charts/cloudnative-pg/values/prod.yaml

```yaml
cloudnative-pg:
  replicaCount: 1

  image:
    repository: ghcr.io/cloudnative-pg/cloudnative-pg
    pullPolicy: IfNotPresent
    tag: ""

  crds:
    create: true

  webhook:
    port: 9443
    mutating:
      create: true
      failurePolicy: Fail
    validating:
      create: true
      failurePolicy: Fail

  config:
    create: true
    clusterWide: true
    secret: false
    data: {}

  maxConcurrentReconciles: 10

  serviceAccount:
    create: true
    name: ""

  rbac:
    create: true
    aggregateClusterRoles: false

  containerSecurityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    runAsUser: 10001
    runAsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
    capabilities:
      drop:
        - "ALL"

  podSecurityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi

  monitoring:
    podMonitorEnabled: false
    grafanaDashboard:
      create: false
```

### Download sub-chart dependencies

```bash
cd homelab-infra-charts/charts/cloudnative-pg
helm dependency build
```

---

## Part 2: PostgreSQL Cluster

A single-node PostgreSQL 16 instance managed by CNPG. The `Cluster` CR tells CNPG what PostgreSQL version, storage, and configuration to use.

Superuser access is **disabled** by default. Use the `app` user for applications, or `kubectl exec` for admin tasks (peer authentication, no password needed).

### charts/postgres-cluster/Chart.yaml

```yaml
apiVersion: v2
name: postgres-cluster
description: Single-node PostgreSQL cluster using CloudNativePG
type: application
version: 1.0.0
```

### charts/postgres-cluster/values/prod.yaml

```yaml
postgres:
  name: homelab-postgres
  namespace: postgres

  instances: 1

  postgresVersion: 16

  storage:
    size: 5Gi
    storageClass: local-path

  resources:
    requests:
      memory: 512Mi
      cpu: "500m"
    limits:
      memory: 1Gi
      cpu: "1"

  parameters:
    max_connections: "200"
    shared_buffers: "256MB"
    effective_cache_size: "768MB"
    maintenance_work_mem: "128MB"
    checkpoint_completion_target: "0.9"
    wal_buffers: "16MB"
    default_statistics_target: "100"
    random_page_cost: "1.1"
    effective_io_concurrency: "200"
    work_mem: "1310kB"
    huge_pages: "off"
    min_wal_size: "1GB"
    max_wal_size: "4GB"

  backup:
    enabled: false

  monitoring:
    enablePodMonitor: false

  service:
    type: LoadBalancer
    loadBalancerIP: 192.168.0.204
    port: 5432
```

### templates/cluster.yaml

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: {{ .Values.postgres.name }}
  namespace: {{ .Values.postgres.namespace }}
spec:
  description: Single-node PostgreSQL cluster for homelab

  imageName: ghcr.io/cloudnative-pg/postgresql:16

  instances: {{ .Values.postgres.instances }}

  enableSuperuserAccess: false

  postgresUID: 26
  postgresGID: 26

  primaryUpdateStrategy: unsupervised

  storage:
    size: {{ .Values.postgres.storage.size }}
    storageClass: {{ .Values.postgres.storage.storageClass }}

  resources:
    requests:
      memory: {{ .Values.postgres.resources.requests.memory }}
      cpu: {{ .Values.postgres.resources.requests.cpu }}
    limits:
      memory: {{ .Values.postgres.resources.limits.memory }}
      cpu: {{ .Values.postgres.resources.limits.cpu }}

  postgresql:
    parameters:
      {{- toYaml .Values.postgres.parameters | nindent 6 }}
```

### templates/service.yaml

```yaml
{{- if .Values.postgres.service }}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.postgres.name }}-external
  namespace: {{ .Values.postgres.namespace }}
  labels:
    cnpg.io/cluster: {{ .Values.postgres.name }}
spec:
  type: {{ .Values.postgres.service.type }}
  {{- if .Values.postgres.service.loadBalancerIP }}
  loadBalancerIP: {{ .Values.postgres.service.loadBalancerIP }}
  {{- end }}
  ports:
    - port: {{ .Values.postgres.service.port }}
      targetPort: 5432
      protocol: TCP
      name: postgres
  selector:
    cnpg.io/cluster: {{ .Values.postgres.name }}
    role: primary
{{- end }}
```

---

## Part 3: ArgoCD Applications

### cloudnative-pg operator (infra-apps/cloudnative-pg.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudnative-pg
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
    targetRevision: HEAD
    path: charts/cloudnative-pg
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: cnpg-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### postgres-cluster (apps/postgres-cluster.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: postgres-cluster
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "10"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
    targetRevision: HEAD
    path: charts/postgres-cluster
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: postgres
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - SkipDryRunOnMissingResource=true
```

---

## Verify Deployment

### Check operator status

```bash
kubectl get pods -n cnpg-system
# NAME                                      READY   STATUS    RESTARTS   AGE
# cnpg-cloudnative-pg-7b9c4d5f8b-k2j5l     1/1     Running   0          2m
```

### Check PostgreSQL cluster

```bash
kubectl get clusters -n postgres
# NAME              AGE     INSTANCES   READY   STATUS                 PRIMARY
# homelab-postgres  5m      1           1       Cluster in healthy state  homelab-postgres-1

kubectl get pods -n postgres
# NAME                    READY   STATUS    RESTARTS   AGE
# homelab-postgres-1      1/1     Running   0          5m
```

### Check external access

```bash
kubectl get svc -n postgres
# NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
# homelab-postgres         ClusterIP      10.x.x.x      <none>          5432/TCP         5m
# homelab-postgres-external LoadBalancer   10.x.x.x      192.168.0.204  5432:xxxxx/TCP   5m
```

### Check app user secret

CNPG auto-creates the `app` user and stores credentials in a secret:

```bash
kubectl get secret homelab-postgres-app -n postgres -o jsonpath='{.data}' | \
  python3 -c "import sys, json, base64; d=json.load(sys.stdin); print('\n'.join(f'{k}: {base64.b64decode(v).decode()}' for k,v in d.items()))"
```

---

## Connect to PostgreSQL

### Connect via LoadBalancer (app user)

```bash
# Get the app user password
kubectl get secret homelab-postgres-app -n postgres -o jsonpath='{.data.password}' | base64 -d

# Connect with pgcli
pgcli -h 192.168.0.204 -p 5432 -U app -d app
```

### Connect via port-forward (app user)

```bash
kubectl port-forward -n postgres svc/homelab-postgres 5432:5432

# In another terminal:
pgcli -h localhost -p 5432 -U app -d app
```

### Admin access via kubectl exec (peer auth, no password)

```bash
kubectl exec -n postgres homelab-postgres-1 -c postgres -- psql -U postgres
```

---

## Creating New Databases and Roles

CNPG provides declarative CRDs for managing databases and roles.

### Create the Role Secret

First, create the secret that will store the role credentials:

```bash
kubectl create secret generic myapp-role-secret -n postgres \
  --from-literal=username=myapp \
  --from-literal=password='secure-password-here'
```

Or as a YAML manifest:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-role-secret
  namespace: postgres
type: kubernetes.io/basic-auth
stringData:
  username: myapp
  password: "secure-password-here"
```

### Create a Role with DatabaseRole CRD

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: DatabaseRole
metadata:
  name: myapp-role
  namespace: postgres
spec:
  cluster:
    name: homelab-postgres
  name: myapp
  login: true
  superuser: false
  createdb: true
  passwordSecret:
    name: myapp-role-secret
```

### Create a Database with Database CRD

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: myapp-db
  namespace: postgres
spec:
  name: myapp
  owner: myapp
  cluster:
    name: homelab-postgres
  extensions:
    - name: pg_trgm
      ensure: present
```

### Apply Everything

```bash
# Create the secret
kubectl create secret generic myapp-role-secret -n postgres \
  --from-literal=username=myapp \
  --from-literal=password='secure-password-here'

# Apply the DatabaseRole and Database CRDs
kubectl apply -f myapp-role.yaml
kubectl apply -f myapp-db.yaml
```

### Connect to the New Database

```bash
# Get the role password
kubectl get secret myapp-role-secret -n postgres -o jsonpath='{.data.password}' | base64 -d

# Connect
pgcli -h 192.168.0.204 -p 5432 -U myapp -d myapp
```

### Grant Additional Permissions (via kubectl exec)

```bash
kubectl exec -n postgres homelab-postgres-1 -c postgres -- psql -U postgres

# Grant schema permissions
GRANT ALL ON SCHEMA public TO myapp;
GRANT ALL PRIVILEGES ON DATABASE myapp TO myapp;

# Grant read-only access to another role
GRANT CONNECT ON DATABASE myapp TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
```

---

## Useful CNPG Commands

```bash
# Check cluster status
kubectl cnpg status homelab-postgres -n postgres

# List all clusters
kubectl cnpg clusters --all-namespaces

# Scale up to 3 instances (edit the values and sync)
kubectl edit cluster homelab-postgres -n postgres

# Restart PostgreSQL
kubectl cnpg restart instance homelab-postgres-1 -n postgres

# List all databases
kubectl exec -n postgres homelab-postgres-1 -c postgres -- \
  psql -U postgres -c "\l"

# List all roles
kubectl exec -n postgres homelab-postgres-1 -c postgres -- \
  psql -U postgres -c "\du"
```

---

## Troubleshooting

### Cluster stuck in "Setting up primary"

The operator is waiting for the PostgreSQL instance to be ready. Check the operator logs:

```bash
kubectl logs -n cnpg-system -l app.kubernetes.io/name=cloudnative-pg --tail=50
```

### PVC pending

If the PostgreSQL pod is stuck in `Pending`, the storage class may not be available:

```bash
kubectl get pvc -n postgres
kubectl get sc local-path
```

### Operator crash loop

Check if CRDs are installed:

```bash
kubectl get crd | grep cnpg
# clusters.postgresql.cnpg.io
# databases.postgresql.cnpg.io
# databaseroles.postgresql.cnpg.io
# backups.postgresql.cnpg.io
# ...
```

### Connection refused from outside

Check if the LoadBalancer is assigned:

```bash
kubectl get svc -n postgres homelab-postgres-external
# EXTERNAL-IP should be 192.168.0.204
```

---

## Summary

| Component | Namespace | Access |
|-----------|-----------|--------|
| CNPG Operator | `cnpg-system` | Cluster-scoped |
| PostgreSQL | `postgres` | Internal: `homelab-postgres-rw.postgres.svc:5432` |
| | | External: `192.168.0.204:5432` |
| App User | `postgres` | `homelab-postgres-app` secret |
| Admin Access | `postgres` | `kubectl exec` (peer auth, no password) |

## References

- [CloudNativePG Documentation](https://cloudnative-pg.io/documentation/)
- [CNPG Helm Charts](https://github.com/cloudnative-pg/charts)
- [CNPG CRD API Reference](https://cloudnative-pg.io/documentation/latest/api_reference/)
- [Declarative Database Management](https://cloudnative-pg.io/documentation/latest/declarative_database_management/)
- [Declarative Role Management](https://cloudnative-pg.io/documentation/latest/declarative_role_management/)
- [PostgreSQL on CNPG](https://cloudnative-pg.io/documentation/latest/postgresql/)
