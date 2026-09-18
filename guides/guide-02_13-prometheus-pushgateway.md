# 02-13: Deploy Prometheus Pushgateway via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy.
- [ ] ArgoCD bootstrapped and running (Guide 2)
- [ ] Root app synced and healthy
- [ ] MetalLB deployed
- [ ] Traefik + cert-manager deployed
- [ ] kube-prometheus-stack deployed (Guide 2-09)
- [ ] CSI NFS driver configured with `nfs-csi-pve2` storage class (Guide 2-06)

## What We're Building

Prometheus Pushgateway for receiving metrics from short-lived jobs, batch processes, and services that cannot be scraped directly by Prometheus.

| Component | Version | Purpose |
|-----------|---------|---------|
| prometheus-pushgateway | v1.11.3 | Receives and stores pushed metrics |
| ServiceMonitor | - | Auto-discovery by Prometheus |
| NFS PVC | 5Gi on `nfs-csi-pve2` | Persistent metric storage |

### How Pushgateway Fits in Your Stack

```
Short-lived jobs, CronJobs, Services
        │
        │  POST /metrics/job/<name>/instance/<id>
        ▼
┌─────────────────────┐
│  Prometheus         │
│  Pushgateway        │◄── Scraps pushgateway every 30s
│  :9091/metrics      │
└─────────────────────┘
        │
        │  Persistent storage (NFS)
        ▼
┌─────────────────────┐
│  Prometheus         │
│  TSDB               │
│  (15d retention)    │
└─────────────────────┘
```

## prometheus-pushgateway Umbrella Chart

> [!NOTE]
> The chart is in the [homelab-infra-charts repo](https://github.com/utkarsh-homelab/homelab-infra-charts).
>
> `git clone git@github.com:utkarsh-homelab/homelab-infra-charts.git`

### charts/prometheus-pushgateway/Chart.yaml

```yaml
apiVersion: v2
name: prometheus-pushgateway
description: Vendored prometheus-pushgateway Helm chart for homelab
type: application
version: 1.0.0
appVersion: v1.11.3
dependencies:
  - name: prometheus-pushgateway
    version: 3.8.0
    repository: https://prometheus-community.github.io/helm-charts
```

### charts/prometheus-pushgateway/values/prod.yaml

```yaml
prometheus-pushgateway:
  replicaCount: 1

  persistence:
    enabled: true
    storageClass: nfs-csi-pve2
    accessModes:
      - ReadWriteOnce
    size: 5Gi

  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 100m
      memory: 128Mi

  serviceMonitor:
    enabled: true
    additionalLabels:
      release: kube-prometheus-stack
    interval: 30s

  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9091"

  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    runAsNonRoot: true
```

### Download sub-chart dependencies

```bash
cd ./homelab-infra-charts

# Add upstream Helm repo (if not already added)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Download sub-chart dependencies
helm dependency build charts/prometheus-pushgateway/
```

## prometheus-pushgateway GitOps App

> [!NOTE]
> The GitOps repo can be found [here](https://github.com/utkarsh-homelab/homelab-gitops)

The `infra-apps/prometheus-pushgateway.yaml` references the infra-charts repo on sync wave 1:

```yaml
# gitops/infra-apps/prometheus-pushgateway.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-pushgateway
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
    path: charts/prometheus-pushgateway
    helm:
      valueFiles:
        - values/prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
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

## Verify

```bash
# Check pods
kubectl get pods -n monitoring -l app=prometheus-pushgateway

# Check service
kubectl get svc -n monitoring -l app=prometheus-pushgateway

# Check PVC
kubectl get pvc -n monitoring | grep pushgateway

# Check ServiceMonitor
kubectl get servicemonitor -n monitoring | grep pushgateway

# Check logs
kubectl logs -n monitoring -l app=prometheus-pushgateway
```

## Access Pushgateway UI

```bash
kubectl port-forward -n monitoring svc/prometheus-pushgateway 9091:9091 &
```

Open `http://localhost:9091` in your browser. You should see the Pushgateway web UI with no metrics initially.

### Via Ingress (LAN)

Access at: `https://pushgateway.uttutu.xyz`

## Verify Pi-hole DNS Records

Add this to Pi-hole Local DNS Records:

| Domain | IP |
|--------|-----|
| `pushgateway.uttutu.xyz` | `192.168.0.200` |

## Test: Push Metrics via Curl

Push a test metric to verify connectivity:

```bash
curl --data-binary @- https://pushgateway.uttutu.xyz/metrics/job/test_job/instance/test_instance <<EOF
# HELP test_requests_total Total test requests
# TYPE test_requests_total counter
test_requests_total{method="GET", status="200"} 42

# HELP test_latency_seconds Request latency
# TYPE test_latency_seconds histogram
test_latency_seconds_bucket{le="0.1"} 10
test_latency_seconds_bucket{le="0.5"} 18
test_latency_seconds_bucket{le="1.0"} 20
test_latency_seconds_bucket{le="+Inf"} 20
test_latency_seconds_sum 5.2
test_latency_seconds_count 20
EOF
```

Verify the metric appeared:

```bash
curl https://pushgateway.uttutu.xyz/metrics | grep test_requests_total
```

You should see output like:

```
# HELP test_requests_total Total test requests
# TYPE test_requests_total counter
test_requests_total{instance="test_instance", job="test_job", method="GET", status="200"} 42
```

## Verify Prometheus is Scraping

1. Open `https://prometheus.uttutu.xyz/targets`
2. Search for `pushgateway` in the targets list
3. Confirm the target state is **UP**
4. Query `test_requests_total` in the Prometheus UI

## Using Pushgateway from Cluster Services

### In-Cluster Service URL

Within the cluster, pushgateway is accessible at:

```
http://prometheus-pushgateway.monitoring:9091
```

Or with full DNS:

```
http://prometheus-pushgateway.monitoring.svc.cluster.local:9091
```

### Job Naming Conventions

Every push **must** include a `job` label. Use consistent naming:

| Use Case | Job Name Pattern |
|----------|------------------|
| CronJob | `cron_<name>` |
| Deployment | `<app_name>` |
| Batch job | `batch_<name>` |
| One-time script | `script_<name>` |

### Grouping Keys

Grouping keys allow multiple pushes to the same job to be distinguished. Common patterns:

```
/metrics/job/<job_name>/instance/<instance_id>
/metrics/job/<job_name>/namespace/<namespace>/pod/<pod_name>
/metrics/job/<job_name>/team/<team_name>
```

> [!IMPORTANT]
> Metrics with different grouping keys are treated as separate jobs in Pushgateway. Choose your grouping keys carefully.

### Method 1: HTTP API (Universal)

Use `curl` or `wget` from any pod in the cluster:

```bash
# From a debug pod
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n default -- /bin/sh

# Inside the pod, push metrics
curl --data-binary @- http://prometheus-pushgateway.monitoring:9091/metrics/job/my_app/instance/pod-123 <<EOF
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET", path="/api/v1/users"} 150
http_requests_total{method="POST", path="/api/v1/users"} 25

# HELP http_request_duration_seconds Request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET", le="0.1"} 120
http_request_duration_seconds_bucket{method="GET", le="0.5"} 145
http_request_duration_seconds_bucket{method="GET", le="1.0"} 149
http_request_duration_seconds_bucket{method="GET", le="+Inf"} 150
http_request_duration_seconds_sum{method="GET"} 12.5
http_request_duration_seconds_count{method="GET"} 150
EOF
```

### Method 2: Python Client Library

```python
from prometheus_client import (
    Counter, Gauge, Histogram, Summary,
    CollectorRegistry, push_to_gateway, delete_from_gateway
)
import time
import socket

# Create a registry for this job
registry = CollectorRegistry()

# Define metrics with labels
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status'],
    registry=registry
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency in seconds',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0],
    registry=registry
)

ACTIVE_JOBS = Gauge(
    'active_jobs_count',
    'Number of currently active jobs',
    ['queue'],
    registry=registry
)

def push_metrics():
    """Push current metrics to pushgateway."""
    instance_id = socket.gethostname()

    push_to_gateway(
        'prometheus-pushgateway.monitoring:9091',
        job='my-python-app',
        registry=registry,
        grouping_key={'instance': instance_id}
    )

def record_request(method, endpoint, status_code, duration):
    """Record a single request's metrics."""
    REQUEST_COUNT.labels(
        method=method,
        endpoint=endpoint,
        status=str(status_code)
    ).inc()

    REQUEST_LATENCY.labels(
        method=method,
        endpoint=endpoint
    ).observe(duration)

def cleanup():
    """Delete metrics from pushgateway on job completion."""
    delete_from_gateway(
        'prometheus-pushgateway.monitoring:9091',
        job='my-python-app',
        grouping_key={'instance': socket.gethostname()}
    )

# Example usage
if __name__ == '__main__':
    try:
        # Record some requests
        record_request('GET', '/api/users', 200, 0.12)
        record_request('GET', '/api/users', 200, 0.08)
        record_request('POST', '/api/users', 201, 0.35)

        # Set active jobs
        ACTIVE_JOBS.labels(queue='default').set(3)

        # Push metrics
        push_metrics()
        print("Metrics pushed successfully")
    except Exception as e:
        print(f"Failed to push metrics: {e}")
    finally:
        # Optional: cleanup on script exit
        # cleanup()
        pass
```

**Requirements:**

```
prometheus_client>=0.14.0
```

**Install:**

```bash
pip install prometheus_client
```

### Method 3: Go Client Library

```go
package main

import (
	"log"
	"net/http"
	"os"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/push"
)

var (
	requestCount = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "Total HTTP requests",
		},
		[]string{"method", "endpoint", "status"},
	)

	requestDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "http_request_duration_seconds",
			Help:    "HTTP request latency",
			Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1.0, 5.0},
		},
		[]string{"method", "endpoint"},
	)
)

func init() {
	prometheus.MustRegister(requestCount, requestDuration)
}

func pushMetrics(jobName, instanceID string) error {
	pusher := push.New("http://prometheus-pushgateway.monitoring:9091", jobName).
		Collector(requestCount).
		Collector(requestDuration).
		Grouping("instance", instanceID)

	if err := pusher.Add(); err != nil {
		return err
	}
	return nil
}

func cleanup(jobName, instanceID string) error {
	return push.New("http://prometheus-pushgateway.monitoring:9091", jobName).
		Grouping("instance", instanceID).
		Delete()
}

func main() {
	instanceID, _ := os.Hostname()
	jobName := "my-go-app"

	// Simulate some requests
	for i := 0; i < 10; i++ {
		requestCount.WithLabelValues("GET", "/api/data", "200").Inc()
		requestDuration.WithLabelValues("GET", "/api/data").Observe(0.15)
	}

	// Push to pushgateway
	if err := pushMetrics(jobName, instanceID); err != nil {
		log.Fatalf("Failed to push metrics: %v", err)
	}
	log.Println("Metrics pushed successfully")

	// Keep running and pushing periodically
	ticker := time.NewTicker(30 * time.Second)
	defer ticker.Stop()

	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	log.Println("Starting server on :8080")
	http.ListenAndServe(":8080", nil)
}
```

**Requirements:**

```go
require (
    github.com/prometheus/client_golang v1.18.0
)
```

### Method 4: Other Languages via HTTP API

Any language that can make HTTP requests can use pushgateway:

**Bash script:**

```bash
#!/bin/bash
PUSHGATEWAY_URL="http://prometheus-pushgateway.monitoring:9091"
JOB_NAME="backup_job"
TIMESTAMP=$(date +%s)

curl --data-binary @- "$PUSHGATEWAY_URL/metrics/job/$JOB_NAME/instance/$(hostname)" <<EOF
# HELP backup_last_success_timestamp_seconds Last successful backup timestamp
# TYPE backup_last_success_timestamp_seconds gauge
backup_last_success_timestamp_seconds $TIMESTAMP
# HELP backup_duration_seconds Backup duration
# TYPE backup_duration_seconds gauge
backup_duration_seconds 125.5
EOF
```

**Node.js:**

```javascript
const client = require('prom-client');

// Create a Registry
const registry = new client.Registry();

// Define metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1.0],
  registers: [registry]
});

async function pushMetrics(jobName, instanceId) {
  const gatewayUrl = 'http://prometheus-pushgateway.monitoring:9091';
  const groups = { job: jobName, instance: instanceId };

  await client.pushgateway.push({
    gatewayUrl,
    jobs: [groups],
    registry
  });
}

// Usage
const timer = httpRequestDuration.startTimer({ method: 'GET', route: '/api' });
// ... do work ...
timer({ status_code: 200 });

pushMetrics('my-node-app', process.env.HOSTNAME || 'local');
```

**Requirements:**

```bash
npm install prom-client
```

## Example: CronJob with Pushgateway

A complete CronJob that runs every 5 minutes, pushes metrics, and cleans up after itself:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: health-check-pusher
  namespace: monitoring
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 60
      template:
        metadata:
          labels:
            app: health-check
        spec:
          serviceAccountName: default
          containers:
          - name: health-check
            image: curlimages/curl:latest
            env:
            - name: PUSHGATEWAY_URL
              value: "http://prometheus-pushgateway.monitoring:9091"
            - name: JOB_NAME
              value: "health_check"
            command:
            - /bin/sh
            - -c
            - |
              # Collect metrics
              POD_COUNT=$(wget -qO- https://kubernetes.default.svc/api/v1/namespaces/default/pods \
                --header="Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
                --no-check-certificate 2>/dev/null | grep -o '"phase"' | wc -l)

              TIMESTAMP=$(date +%s)

              # Push metrics
              curl --data-binary @- "$PUSHGATEWAY_URL/metrics/job/$JOB_NAME/instance/health-checker" <<EOF
              # HELP kubernetes_pod_count Total pods in namespace
              # TYPE kubernetes_pod_count gauge
              kubernetes_pod_count $POD_COUNT
              # HELP health_check_last_run_timestamp_seconds Last check timestamp
              # TYPE health_check_last_run_timestamp_seconds gauge
              health_check_last_run_timestamp_seconds $TIMESTAMP
              # HELP health_check_success_total Successful checks
              # TYPE health_check_success_total counter
              health_check_success_total 1
              EOF

              echo "Pushed metrics successfully"
          restartPolicy: OnFailure
```

## Example: Deployment with Periodic Push

A Deployment that pushes metrics every 30 seconds using a sidecar pattern:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-with-metrics
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 8080

      - name: metrics-pusher
        image: curlimages/curl:latest
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            # Collect your app's metrics here
            TIMESTAMP=$(date +%s)
            ACTIVE=$(cat /tmp/active_connections 2>/dev/null || echo 0)

            curl --data-binary @- \
              http://prometheus-pushgateway.monitoring:9091/metrics/job/my_app/instance/$(hostname) \
              <<EOF
            # HELP app_active_connections Active connections
            # TYPE app_active_connections gauge
            app_active_connections $ACTIVE
            # HELP app_last_heartbeat_timestamp Last heartbeat
            # TYPE app_last_heartbeat_timestamp gauge
            app_last_heartbeat_timestamp $TIMESTAMP
            EOF

            sleep 30
          done
```

## Grafana Dashboard Import

Import a pre-built Pushgateway dashboard:

1. Open `https://grafana.uttutu.xyz`
2. Go to **Dashboards > Import**
3. Enter dashboard ID: **13770** (Prometheus Pushgateway)
4. Select your Prometheus datasource
5. Click **Import**

The dashboard will show:
- Push rate (metrics/second)
- Number of jobs
- Push duration
- Metric cardinality

## Troubleshooting

### Metrics Not Appearing in Prometheus

**Check ServiceMonitor exists:**

```bash
kubectl get servicemonitor -n monitoring -o yaml | grep -A 20 pushgateway
```

**Check Prometheus targets:**

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Open http://localhost:9090/targets and search for pushgateway
```

**Verify Prometheus can reach pushgateway:**

```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n monitoring -- \
  curl -s http://prometheus-pushgateway.monitoring:9091/metrics | head -20
```

### PVC Stuck in Pending

**Check PVC status:**

```bash
kubectl get pvc -n monitoring | grep pushgateway
kubectl describe pvc -n monitoring <pvc-name>
```

**Verify nfs-csi-pve2 StorageClass exists:**

```bash
kubectl get storageclass nfs-csi-pve2
```

### Pushgateway Pod CrashLoopBackOff

**Check logs:**

```bash
kubectl logs -n monitoring -l app=prometheus-pushgateway --previous
```

**Common issues:**
- Permission denied on PVC mount → Check securityContext (uid 1000)
- NFS server unreachable → Verify NFS server at `192.168.0.110` is accessible

### Metric Labels Not Updating

Pushgateway does **not** update existing metrics by default. You must either:

1. **Delete the metric first** (recommended):

```bash
curl -X DELETE https://pushgateway.uttutu.xyz/metrics/job/my_job/instance/my_instance
```

2. **Use `--push-replace` flag** (requires pushgateway v1.7.0+):

```bash
curl --data-binary @- --push-replace \
  https://pushgateway.uttutu.xyz/metrics/job/my_job/instance/my_instance \
  <<EOF
# HELP my_metric Updated metric value
# TYPE my_metric gauge
my_metric 100
EOF
```

### High Memory Usage

Pushgateway stores all metrics in memory. If memory usage is high:

1. Check metric cardinality (too many label combinations)
2. Implement metric cleanup using the `DELETE` API
3. Increase memory limits in `values/prod.yaml`

## Summary

| Component | Namespace | Service URL |
|-----------|-----------|-------------|
| Pushgateway UI | monitoring | `https://pushgateway.uttutu.xyz` |
| Pushgateway (in-cluster) | monitoring | `http://prometheus-pushgateway.monitoring:9091` |
| Pushgateway (DNS) | monitoring | `http://prometheus-pushgateway.monitoring.svc.cluster.local:9091` |

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/metrics/job/<name>` | Push metrics for a job |
| `POST` | `/metrics/job/<name>/instance/<id>` | Push metrics with instance grouping |
| `GET` | `/metrics` | List all pushed metrics |
| `DELETE` | `/metrics/job/<name>` | Delete all metrics for a job |
| `DELETE` | `/metrics/job/<name>/instance/<id>` | Delete specific instance metrics |
| `GET` | `/-/healthy` | Health check endpoint |
| `GET` | `/-/ready` | Readiness check endpoint |

## References

- [Prometheus Pushgateway GitHub](https://github.com/prometheus/pushgateway)
- [Pushgateway Helm Chart](https://artifacthub.io/packages/helm/prometheus-community/prometheus-pushgateway)
- [Prometheus Client Libraries](https://prometheus.io/docs/instrumenting/clientlibs/)
- [Pushgateway Best Practices](https://prometheus.io/docs/practices/pushing/)
