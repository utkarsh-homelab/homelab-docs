# 02-10: Deploy Strimzi Kafka via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy
- [ ] ArgoCD bootstrapped and running [Guide - Manual](./guide-02_04-argocd-bootstrap-and-gitops-setup-manual.md) or [Guide - Automated](./guide-02_05-automating-argocd-bootstrap.md)
- [ ] Root app synced and healthy
- [ ] MetalLB deployed [Guide](./guide-02_07-metallb.md)
- [ ] CSI Driver NFS deployed [Guide](./guide-02_06-csi-driver-nfs.md)

## What We're Building

| Component | Value |
|-----------|-------|
| Strimzi Operator | Latest (Helm chart) |
| Kafka Version | 4.x (KRaft mode, no ZooKeeper) |
| Broker Count | 1 (single combined controller+broker) |
| Storage | Local-path provisioner (5Gi) |
| Internal Listener | `plain` on port 9092 |
| External Listener | `loadbalancer` on port 9094 via MetalLB |
| Namespace | `kafka` |

> [!NOTE]
> Strimzi requires **block storage** — NFS is not supported. We deploy a local-path provisioner first, then Kafka on top of it.

## Why KRaft?

KRaft (Kafka Raft) replaces ZooKeeper for metadata management. Since Kafka 4.0, ZooKeeper is fully removed. KRaft simplifies the deployment by eliminating a separate ZooKeeper ensemble and improves startup/recovery time.

---

## Part 1: Local-Path Storage Provisioner

Strimzi requires block storage. Since our cluster only has NFS, we deploy the `rancher.io/local-path` provisioner to provide dynamic PVC provisioning on local node disks.

### charts/local-path-provisioner/Chart.yaml

```yaml
apiVersion: v2
name: local-path-provisioner
description: Local path dynamic provisioner for Strimzi Kafka
type: application
version: 1.0.0
appVersion: 0.0.37
dependencies:
  - name: local-path-provisioner
    version: 0.0.37
    repository: "oci://ghcr.io/rancher/local-path-provisioner/charts"
```

### charts/local-path-provisioner/values/prod.yaml

```yaml
local-path-provisioner:
  enabled: true

  replicaCount: 1

  image:
    repository: rancher/local-path-provisioner
    tag: v0.0.37
    pullPolicy: IfNotPresent

  helperImage:
    repository: busybox
    tag: latest

  storageClass:
    create: true
    name: local-path
    defaultClass: false
    reclaimPolicy: Delete
    volumeBindingMode: WaitForFirstConsumer
    allowVolumeExpansion: true

  helperPod:
    image:
      repository: busybox
      tag: "latest"

  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 100m
      memory: 128Mi

  nodePathMap:
    - node: DEFAULT_PATH_FOR_NON_LISTED_NODES
      paths:
        - /opt/local-path-provisioner
```

> [!NOTE]
> `defaultClass: false` keeps NFS CSI (`nfs-csi`) as the default StorageClass. The `local-path` StorageClass is available for Kafka PVCs that explicitly reference it. This way your existing workloads continue using NFS while Kafka gets local block storage.

### Download sub-chart dependencies

```bash
cd ./homelab-infra-charts

# Download sub-chart dependencies (local-path-provisioner uses OCI, no repo add needed)
helm dependency build charts/local-path-provisioner/
```

### GitOps Application

Create `infra-apps/local-path-provisioner.yaml` in the [homelab-gitops repo](https://github.com/utkarsh-homelab/homelab-gitops):

```yaml
# homelab-gitops/infra-apps/local-path-provisioner.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: local-path-provisioner
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
    path: charts/local-path-provisioner
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: local-path-storage
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Part 2: Strimzi Kafka Operator

The Cluster Operator watches `Kafka` custom resources and manages the full lifecycle — creating StatefulSets, Services, ConfigMaps, and more.

### charts/strimzi-kafka/Chart.yaml

```yaml
apiVersion: v2
name: strimzi-kafka
description: Vendored Strimzi Kafka operator Helm chart for uttutu homelab
type: application
version: 1.0.0
appVersion: 0.46.1
dependencies:
  - name: strimzi-kafka-operator
    version: 0.46.1
    repository: https://strimzi.github.io/charts
```

> [!NOTE]
> Check [Strimzi releases](https://github.com/strimzi/strimzi-kafka-operator/releases) for the latest version. The operator version should match or be compatible with your target Kafka version.

### charts/strimzi-kafka/values/prod.yaml

```yaml
strimzi-kafka-operator:
  watchNamespaces: []
  watchAllNamespaces: true

  replicas: 1

  resources:
    requests:
      cpu: 200m
      memory: 384Mi
    limits:
      cpu: 1000m
      memory: 384Mi

  logLevel: INFO
  logFormat: json

  featureGates: ""

  operationTimeoutMs: 300000

  operator:
    resources:
      requests:
        cpu: 200m
        memory: 384Mi
      limits:
        cpu: 1000m
        memory: 384Mi

  topicOperator:
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 256Mi

  userOperator:
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 256Mi

  featureGates: ""
```

> [!NOTE]
> `watchAllNamespaces: true` allows the operator to manage Kafka clusters in any namespace. This is simpler than namespace-scoped watching for a single-cluster homelab.

### Download sub-chart dependencies and CRDs

```bash
cd ./homelab-infra-charts

# Add upstream Helm repos
helm repo add strimzi https://strimzi.github.io/charts

# Download sub-chart dependencies 
helm dependency build charts/strimzi-kafka/

# Download Strimzi CRDs (the operator chart does not bundle them)
mkdir -p charts/strimzi-kafka/manifests
curl -sL https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.46.1/strimzi-crds-0.46.1.yaml \
  -o charts/strimzi-kafka/manifests/strimzi-crds.yaml
```

### GitOps Application

Create `infra-apps/strimzi-kafka.yaml` in the [homelab-gitops repo](https://github.com/utkarsh-homelab/homelab-gitops):

```yaml
# homelab-gitops/infra-apps/strimzi-kafka.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: strimzi-kafka
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
      targetRevision: HEAD
      path: charts/strimzi-kafka
      helm:
        valueFiles:
          - values/prod.yaml
    - repoURL: https://github.com/utkarsh-homelab/homelab-infra-charts
      targetRevision: HEAD
      path: charts/strimzi-kafka/manifests
      directory:
        recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: kafka
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> [!NOTE]
> The Strimzi operator Helm chart does **not** bundle CRDs (same issue as ArgoCD). We use multi-source to sync both the Helm chart and the raw CRD manifests from `charts/strimzi-kafka/manifests/`. The CRDs are downloaded from the [Strimzi releases](https://github.com/strimzi/strimzi-kafka-operator/releases) as `strimzi-crds-<version>.yaml`.

---

## Part 3: Kafka Cluster (Single-Node KRaft)

This is the actual Kafka workload — a single broker running in KRaft mode with both controller and broker roles combined.

### charts/kafka-cluster/Chart.yaml

This is a standalone chart (no upstream dependency) that contains the `Kafka` and `KafkaNodePool` custom resources:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kafka-test-client
  namespace: kafka
spec:
  containers:
    - name: kafka
      image: quay.io/strimzi/kafka:latest-kafka-4.0.0
      command:
        - /bin/sh
        - -c
        - |
          export PATH=$PATH:/opt/kafka/bin
          # Wait for Kafka to be ready
          echo "Waiting for Kafka broker..."
          until kafka-topics.sh --bootstrap-server homelab-kafka-kafka-bootstrap:9092 --list; do
            sleep 2
          done
          echo "Kafka is ready!"

          # Create a topic
          kafka-topics.sh --bootstrap-server homelab-kafka-kafka-bootstrap:9092 \
            --create --topic test-topic --partitions 1 --replication-factor 1

          # Produce messages
          echo "Hello from Strimzi Kafka!" | kafka-console-producer.sh \
            --bootstrap-server homelab-kafka-kafka-bootstrap:9092 --topic test-topic

          echo "Message 2" | kafka-console-producer.sh \
            --bootstrap-server homelab-kafka-kafka-bootstrap:9092 --topic test-topic

          # Consume messages
          echo "--- Consuming messages ---"
          kafka-console-consumer.sh \
            --bootstrap-server homelab-kafka-kafka-bootstrap:9092 \
            --topic test-topic --from-beginning --timeout-ms 10000

          echo "Done!"
```

### charts/kafka-cluster/values/prod.yaml

```yaml
kafka:
  name: homelab-kafka
  namespace: kafka

  # Single-node KRaft: combined controller+broker role
  kafkaNodePool:
    name: pooled
    replicas: 1
    storage:
      type: persistent-claim
      size: 5Gi
      class: local-path
      deleteClaim: false
    resources:
      requests:
        memory: 2Gi
        cpu: "1"
      limits:
        memory: 2Gi
        cpu: "2"
    jvmOptions:
      "-Xms": "1g"
      "-Xmx": "1g"

  # Broker configuration
  config:
    # Single-broker replication (safe defaults even with 1 broker)
    offsets.topic.replication.factor: 1
    transaction.state.log.replication.factor: 1
    transaction.state.log.min.isr: 1
    default.replication.factor: 1
    min.insync.replicas: 1
    # Log retention
    log.retention.hours: 168
    log.retention.bytes: 1073741824  # 1GB per partition
    # Performance
    num.partitions: 1
    num.network.threads: 4
    num.io.threads: 8
    socket.send.buffer.bytes: 102400
    socket.receive.buffer.bytes: 102400
    socket.request.max.bytes: 104857600

  # Listeners
  listeners:
    - name: plain
      port: 9092
      type: internal
      tls: false
    - name: tls
      port: 9093
      type: internal
      tls: true
    - name: external
      port: 9094
      type: loadbalancer
      tls: false
      configuration:
        bootstrap:
          loadBalancerIP: 192.168.0.202

  # Entity Operator (Topic + User operators)
  entityOperator:
    topicOperator: {}
    userOperator: {}

  # Disable ZooKeeper (KRaft mode)
  zookeeper: {}
```

> [!CAUTION]
> Update `loadBalancerIP: 192.168.0.202` to a free IP in your MetalLB pool (`192.168.0.200-249`). Check that the IP is not already in use:
>
> ```bash
> kubectl get svc -A -o json | jq -r '.items[].status.loadBalancer.ingress[].ip' | grep 192.168.0
> ```

> [!NOTE]
> With a single broker, replication factors of 1 are necessary — there are no other brokers to replicate to. For production clusters with 3+ brokers, increase these to 3 and set `min.insync.replicas: 2`.

### charts/kafka-cluster/templates/kafkanodepool.yaml

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: {{ .Values.kafka.kafkaNodePool.name }}
  namespace: {{ .Values.kafka.namespace }}
  labels:
    strimzi.io/cluster: {{ .Values.kafka.name }}
spec:
  replicas: {{ .Values.kafka.kafkaNodePool.replicas }}
  roles:
    - controller
    - broker
  storage:
    type: {{ .Values.kafka.kafkaNodePool.storage.type }}
    size: {{ .Values.kafka.kafkaNodePool.storage.size }}
    storageClass: {{ .Values.kafka.kafkaNodePool.storage.class }}
    deleteClaim: {{ .Values.kafka.kafkaNodePool.storage.deleteClaim }}
  resources:
    requests:
      memory: {{ .Values.kafka.kafkaNodePool.resources.requests.memory }}
      cpu: {{ .Values.kafka.kafkaNodePool.resources.requests.cpu | quote }}
    limits:
      memory: {{ .Values.kafka.kafkaNodePool.resources.limits.memory }}
      cpu: {{ .Values.kafka.kafkaNodePool.resources.limits.cpu | quote }}
  jvmOptions:
    "-Xms": {{ .Values.kafka.kafkaNodePool.jvmOptions."-Xms" | quote }}
    "-Xmx": {{ .Values.kafka.kafkaNodePool.jvmOptions."-Xmx" | quote }}
```

### charts/kafka-cluster/templates/kafka.yaml

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: {{ .Values.kafka.name }}
  namespace: {{ .Values.kafka.namespace }}
  annotations:
    strimzi.io/pod-recovery: enabled
    strimzi.io/kraft: enabled
    strimzi.io/node-pools: enabled
spec:
  kafka:
    version: 4.0.0
    listeners:
      {{- toYaml .Values.kafka.listeners | nindent 6 }}
    config:
      {{- toYaml .Values.kafka.config | nindent 6 }}
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

> [!NOTE]
> The `strimzi.io/kraft: enabled` and `strimzi.io/node-pools: enabled` annotations are **required** for Strimzi 0.46+. Without them, the operator rejects the Kafka resource. In KRaft mode, `storage`, `replicas`, `resources`, and `jvmOptions` must only be defined in the `KafkaNodePool` CR — not in the `Kafka` CR. The `Kafka` CR only holds cluster-wide config (listeners, version, entityOperator).

### Download sub-chart dependencies

This is a standalone chart with no external dependencies, so no `helm dependency build` is needed.

### GitOps Application

Create `infra-apps/kafka-cluster.yaml` in the [homelab-gitops repo](https://github.com/utkarsh-homelab/homelab-gitops):

```yaml
# homelab-gitops/infra-apps/kafka-cluster.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kafka-cluster
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
    path: charts/kafka-cluster
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: kafka
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
      - SkipDryRunOnMissingResource=true
```

> [!NOTE]
> `SkipDryRunOnMissingResource=true` is required because ArgoCD renders all manifests before syncing. Without this, it fails when it encounters `Kafka` and `KafkaNodePool` CRDs that don't exist yet (they're created by the Strimzi operator at wave 2). This option tells ArgoCD to skip client-side validation for unknown resource types.

---

## Sync Wave Summary

| Wave | Component | Namespace | Purpose |
|------|-----------|-----------|---------|
| 1 | local-path-provisioner | `local-path-storage` | Block storage for Kafka PVCs |
| 2 | strimzi-kafka (operator) | `kafka` | Strimzi Cluster Operator |
| 10 | kafka-cluster (workload) | `kafka` | Single-node KRaft Kafka |

> [!NOTE]
> Sync wave **10** ensures the Strimzi operator (wave 2) is fully running before the `Kafka` CR is applied. The `SkipDryRunOnMissingResource` annotation is still needed because ArgoCD renders manifests before sync ordering takes effect.

## Resource Budget

| Component | RAM | CPU |
|-----------|-----|-----|
| local-path-provisioner | ~64Mi | ~50m |
| Strimzi Cluster Operator | ~384Mi | ~200m |
| Kafka broker (single, KRaft) | ~2Gi | ~1 core |
| **Total** | **~2.5GB** | **~1.25 cores** |

Well within the 18GB total cluster capacity.

---

## Trigger Sync

```bash
argocd app sync root
```

The sync chain will be: `infra-root` → `local-path-provisioner` (wave 1) → `strimzi-kafka` (wave 2) → `kafka-cluster` (wave 10).

## Verify

```bash
# Check operator is running
kubectl get pods -n kafka

# Check Kafka cluster is being created
kubectl get kafka -n kafka

# Check KafkaNodePool
kubectl get kafkanodepool -n kafka

# Wait for Kafka to be ready
kubectl wait kafka/homelab-kafka --for=condition=Ready --timeout=600s -n kafka

# Check the external listener
kubectl get svc -n kafka
```

Expected output for `kubectl get svc -n kafka`:

```
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
homelab-kafka-external     LoadBalancer   10.x.x.x        192.168.0.202   9094:xxxxx/TCP
homelab-kafka-kafka-bootstrap    ClusterIP      10.x.x.x        <none>          9092/TCP,9093/TCP
homelab-kafka-brokers      ClusterIP      None             <none>          9090/TCP
```

## Test with a Topic

### Create a topic

```yaml
# test-topic.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: test-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: homelab-kafka
spec:
  partitions: 1
  replicas: 1
  config:
    retention.ms: 3600000
    cleanup.policy: delete
```

```bash
kubectl apply -f test-topic.yaml
kubectl get kafkatopic -n kafka
```

### Produce and consume from within the cluster

```yaml
# kafka-test-client.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kafka-test-client
  namespace: kafka
spec:
  containers:
    - name: kafka
      image: quay.io/strimzi/kafka:latest-kafka-4.0.0
      command:
        - /bin/sh
        - -c
        - |
          export PATH=$PATH:/opt/kafka/bin
          # Wait for Kafka to be ready
          echo "Waiting for Kafka broker..."
          until kafka-topics.sh --bootstrap-server homelab-kafka-kafka-bootstrap:9092 --list; do
            sleep 2
          done
          echo "Kafka is ready!"

          # Create a topic
          kafka-topics.sh --bootstrap-server homelab-kafka-kafka-bootstrap:9092 \
            --create --topic test-topic --partitions 1 --replication-factor 1

          # Produce messages
          echo "Hello from Strimzi Kafka!" | kafka-console-producer.sh \
            --bootstrap-server homelab-kafka-kafka-bootstrap:9092 --topic test-topic

          echo "Message 2" | kafka-console-producer.sh \
            --bootstrap-server homelab-kafka-kafka-bootstrap:9092 --topic test-topic

          # Consume messages
          echo "--- Consuming messages ---"
          kafka-console-consumer.sh \
            --bootstrap-server homelab-kafka-kafka-bootstrap:9092 \
            --topic test-topic --from-beginning --timeout-ms 10000

          echo "Done!"
```

```bash
kubectl apply -f kafka-test-client.yaml
kubectl logs -f kafka-test-client -n kafka
```

Expected output:

```
Waiting for Kafka broker...
Kafka is ready!
--- Consuming messages ---
Hello from Strimzi Kafka!
Message 2
Done!
```

Clean up:

```bash
kubectl delete pod kafka-test-client -n kafka
kubectl delete kafkatopic test-topic -n kafka
```

## Access Kafka Externally

### From your laptop

The external listener exposes Kafka at `192.168.0.202:9094`. You can connect using any Kafka client:

```bash
# Using kafka-console-consumer from a container
kubectl run kafka-client --rm -it --restart=Never \
  --image=quay.io/strimzi/kafka:latest-kafka-4.0.0 \
  -- /bin/sh -c "
    export PATH=\$PATH:/opt/kafka/bin
    kafka-console-producer.sh \
      --bootstrap-server 192.168.0.202:9094 \
      --topic test-topic
  "
```

```bash
# Or from your laptop if you have kafka CLI tools installed (open producer in one console and consumer in another)

kafka-console-producer.sh --bootstrap-server 192.168.0.202:9094 --topic test-topic

kafka-console-consumer.sh --bootstrap-server 192.168.0.202:9094 --topic test-topic --from-beginning
```

### TLS External Access

For production use, add TLS and authentication to the external listener:

```yaml
# In values/prod.yaml, update the external listener:
listeners:
  - name: external
    port: 9094
    type: loadbalancer
    tls: true
    authentication:
      type: tls
    configuration:
      bootstrap:
        loadBalancerIP: 192.168.0.202
```

Then create a `KafkaUser` with TLS authentication:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-app
  namespace: kafka
  labels:
    strimzi.io/cluster: homelab-kafka
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: "*"
        operation: All
```

The client certificate and key will be stored in a Secret that you can mount in your application pods.

---

## Troubleshooting

### DNS Resolution Failed

If you see `Failed to resolve server homelab-kafka-bootstrap:9092`, the service name is wrong. Strimzi names the bootstrap service `{cluster-name}-kafka-bootstrap`, not `{cluster-name}-bootstrap`.

```bash
# Check the correct service name
kubectl get svc -n kafka
# Should show: homelab-kafka-kafka-bootstrap (not homelab-kafka-bootstrap)
```

### Image Pull Errors

The `quay.io/strimzi/test-container` images require authentication. Use the `quay.io/strimzi/kafka` image instead:

```yaml
image: quay.io/strimzi/kafka:latest-kafka-4.0.0
```

### KRaft Mode Errors

If you see `nonMigratedCluster || !kraftEnabled || !nodePoolsEnabled`, ensure your Kafka CR has:

```yaml
metadata:
  annotations:
    strimzi.io/kraft: enabled
    strimzi.io/node-pools: enabled
```

Also ensure `storage`, `replicas`, `resources`, and `jvmOptions` are in the KafkaNodePool CR, not the Kafka CR.

### MetalLB IP Conflict

If you get "can't change sharing key" errors, don't set `loadBalancerIP` for both `bootstrap` and `brokers` listeners. For single-broker deployments, only configure the `bootstrap` LoadBalancer IP.

---

## Summary

| Component | Namespace | Access |
|-----------|-----------|--------|
| Strimzi Operator | `kafka` | Cluster-scoped (watches all namespaces) |
| Kafka Cluster | `kafka` | Internal: `homelab-kafka-kafka-bootstrap:9092` |
| | | External: `192.168.0.202:9094` |
| Local-Path Storage | `local-path-storage` | StorageClass `local-path` (default) |

## References

- [Strimzi Documentation](https://strimzi.io/docs/operators/latest/)
- [Strimzi Kafka CRD API Reference](https://strimzi.io/docs/operators/latest/configuring.html)
- [Strimzi Quickstart](https://strimzi.io/quickstarts/)
- [KRaft Documentation](https://kafka.apache.org/documentation/#kraft)
- [rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)
- [Strimzi Helm Chart](https://github.com/strimzi/strimzi-kafka-operator/tree/main/helm-charts/helm3/strimzi-kafka-operator)
