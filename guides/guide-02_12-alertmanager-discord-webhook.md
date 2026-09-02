# 02-12: Alertmanager Discord Webhook

## Prerequisites

- [ ] Kubernetes cluster running and healthy.
- [ ] ArgoCD bootstrapped and running (Guide 2)
- [ ] kube-prometheus-stack deployed (Guide 02-09)
- [ ] A Discord webhook URL (see [Discord Docs](https://discord.com/developers/docs/resources/webhook))

## What We're Building

A Go service that receives Alertmanager webhook payloads and forwards filtered, formatted alerts to Discord.

| Component | Purpose |
|-----------|---------|
| Go webhook service | Translates Alertmanager alerts to Discord embeds |
| ConfigMap | Configurable alert filtering (namespace, labels, severity) |
| Kubernetes Deployment | Runs the service in the `monitoring` namespace |
| Alertmanager config | Routes all alerts to the webhook service |

## Architecture

```
Prometheus → Alertmanager → webhook-service → Discord
                              (Go)
                              ↓
                          ConfigMap
                        (filter rules)
```

## Project Structure

```
alertmanager-discord-webhook/
├── cmd/
│   └── server/
│       └── main.go              # Application entry point
├── internal/
│   ├── alertmanager/
│   │   └── types.go             # Alertmanager webhook payload types
│   ├── config/
│   │   └── config.go            # Configuration loading and alert filtering
│   ├── discord/
│   │   ├── types.go             # Discord webhook payload types
│   │   ├── client.go            # Discord HTTP client
│   │   └── formatter.go         # Alert → Discord embed conversion
│   └── server/
│       └── server.go            # HTTP handlers
├── Dockerfile                   # Multi-stage Docker build
├── go.mod
├── go.sum
└── README.md
```

## Step 1: Create the Go Service

The service lives in `alertmanager-discord-webhook/`.

### internal/config/config.go

```go
package config

import (
	"os"

	"github.com/utkarsh-homelab/alertmanager-discord-webhook/internal/alertmanager"
	"gopkg.in/yaml.v2"
)

type Config struct {
	Namespaces        []string          `yaml:"namespaces"`
	Labels            map[string]string `yaml:"labels"`
	Alertnames        []string          `yaml:"alertnames"`
	ExcludeAlertnames []string          `yaml:"excludeAlertnames"`
	MinSeverity       string            `yaml:"minSeverity"`
}

var severityOrder = map[string]int{
	"info":     1,
	"warning":  2,
	"critical": 3,
}

func Load(path string) (*Config, error) {
	if path == "" {
		return &Config{}, nil
	}

	data, err := os.ReadFile(path)
	if err != nil {
		if os.IsNotExist(err) {
			return &Config{}, nil
		}
		return nil, err
	}

	var cfg Config
	if err := yaml.Unmarshal(data, &cfg); err != nil {
		return nil, err
	}

	return &cfg, nil
}

func (c *Config) ShouldSend(alert alertmanager.Alert) bool {
	if !c.matchNamespaces(alert) {
		return false
	}
	if !c.matchLabels(alert) {
		return false
	}
	if !c.matchAlertnames(alert) {
		return false
	}
	if !c.matchSeverity(alert) {
		return false
	}
	return true
}

func (c *Config) matchNamespaces(alert alertmanager.Alert) bool {
	if len(c.Namespaces) == 0 {
		return true
	}
	namespace := alert.Labels["namespace"]
	for _, ns := range c.Namespaces {
		if ns == namespace {
			return true
		}
	}
	return false
}

func (c *Config) matchLabels(alert alertmanager.Alert) bool {
	if len(c.Labels) == 0 {
		return true
	}
	for k, v := range c.Labels {
		if alert.Labels[k] != v {
			return false
		}
	}
	return true
}

func (c *Config) matchAlertnames(alert alertmanager.Alert) bool {
	alertname := alert.Labels["alertname"]

	for _, exclude := range c.ExcludeAlertnames {
		if exclude == alertname {
			return false
		}
	}

	if len(c.Alertnames) == 0 {
		return true
	}
	for _, name := range c.Alertnames {
		if name == alertname {
			return true
		}
	}
	return false
}

func (c *Config) matchSeverity(alert alertmanager.Alert) bool {
	if c.MinSeverity == "" {
		return true
	}
	severity := alert.Labels["severity"]
	if severity == "" {
		return true
	}
	minLevel, ok := severityOrder[c.MinSeverity]
	if !ok {
		return true
	}
	level, ok := severityOrder[severity]
	if !ok {
		return false
	}
	return level >= minLevel
}
```

### cmd/server/main.go

```go
package main

import (
	"log"
	"net/http"
	"os"

	"github.com/utkarsh-homelab/alertmanager-discord-webhook/internal/config"
	"github.com/utkarsh-homelab/alertmanager-discord-webhook/internal/discord"
	"github.com/utkarsh-homelab/alertmanager-discord-webhook/internal/server"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	discordWebhookURL := os.Getenv("DISCORD_WEBHOOK_URL")
	if discordWebhookURL == "" {
		log.Fatal("DISCORD_WEBHOOK_URL environment variable is required")
	}

	configPath := os.Getenv("CONFIG_PATH")
	cfg, err := config.Load(configPath)
	if err != nil {
		log.Fatalf("Failed to load config: %v", err)
	}
	log.Printf("Loaded config: namespaces=%v, labels=%v, alertnames=%v, excludeAlertnames=%v, minSeverity=%s",
		cfg.Namespaces, cfg.Labels, cfg.Alertnames, cfg.ExcludeAlertnames, cfg.MinSeverity)

	discordClient := discord.NewClient(discordWebhookURL)
	srv := server.New(discordClient, cfg)

	mux := http.NewServeMux()
	mux.HandleFunc("/webhook", srv.HandleWebhook)
	mux.HandleFunc("/healthz", srv.HandleHealthz)

	log.Printf("Starting server on :%s", port)
	if err := http.ListenAndServe(":"+port, mux); err != nil {
		log.Fatal(err)
	}
}
```

### internal/server/server.go

```go
package server

import (
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"

	"github.com/utkarsh-homelab/alertmanager-discord-webhook/internal/alertmanager"
	"github.com/utkarsh-homelab/alertmanager-discord-webhook/internal/config"
	"github.com/utkarsh-homelab/alertmanager-discord-webhook/internal/discord"
)

type Server struct {
	discordClient *discord.Client
	config        *config.Config
}

func New(discordClient *discord.Client, cfg *config.Config) *Server {
	return &Server{
		discordClient: discordClient,
		config:        cfg,
	}
}

func (s *Server) HandleWebhook(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	body, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, "Failed to read body", http.StatusBadRequest)
		return
	}
	defer r.Body.Close()

	var msg alertmanager.Message
	if err := json.Unmarshal(body, &msg); err != nil {
		http.Error(w, "Failed to parse JSON", http.StatusBadRequest)
		return
	}

	filtered := s.filterAlerts(msg.Alerts)
	if len(filtered) == 0 {
		w.WriteHeader(http.StatusOK)
		return
	}

	msg.Alerts = filtered
	embeds := discord.FormatAlerts(msg)

	payload := discord.WebhookPayload{
		Username: "Alertmanager",
		Embeds:   embeds,
	}

	if err := s.discordClient.Send(payload); err != nil {
		log.Printf("Failed to send to Discord: %v", err)
		http.Error(w, "Failed to send to Discord", http.StatusInternalServerError)
		return
	}

	log.Printf("Sent %d alert(s) to Discord (status=%s, group=%s)", len(embeds), msg.Status, msg.GroupKey)
	w.WriteHeader(http.StatusOK)
}

func (s *Server) filterAlerts(alerts []alertmanager.Alert) []alertmanager.Alert {
	if s.config == nil {
		return alerts
	}

	var filtered []alertmanager.Alert
	for _, alert := range alerts {
		if s.config.ShouldSend(alert) {
			filtered = append(filtered, alert)
		}
	}
	return filtered
}

func (s *Server) HandleHealthz(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "ok")
}
```

## Step 2: Build and Push the Docker Image

```bash
cd alertmanager-discord-webhook

docker build -t docker.io/uttutu/alertmanager-discord-webhook:latest .
docker push docker.io/uttutu/alertmanager-discord-webhook:latest
```

## Step 3: Create the Discord Webhook Secret

> [!WARNING]
> This command is run directly on the cluster and NEVER committed to Git.

```bash
kubectl create secret generic alertmanager-discord-webhook \
  --namespace monitoring \
  --from-literal=discord-webhook-url='https://discord.com/api/webhooks/YOUR_WEBHOOK_URL'
```

Replace `YOUR_WEBHOOK_URL` with your actual Discord webhook URL.

To create a Discord webhook:
1. Go to your Discord server settings
2. Navigate to Integrations > Webhooks
3. Click "New Webhook"
4. Name it, select the channel, and copy the URL

## Step 4: Helm Chart

The chart lives in `homelab-infra-charts/charts/alertmanager-discord-webhook/`.

### values/prod.yaml

```yaml
replicaCount: 1

image:
  repository: docker.io/uttutu/alertmanager-discord-webhook
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

existingSecret: alertmanager-discord-webhook

# Alert filtering configuration
config:
  # Only send alerts from these namespaces (empty = all namespaces)
  namespaces: []

  # Only send alerts matching these labels (AND logic)
  labels: {}

  # Only send alerts with these names (empty = all alerts)
  alertnames: []

  # Exclude alerts matching these names
  excludeAlertnames: []

  # Minimum severity: info, warning, critical (empty = all)
  minSeverity: ""

resources:
  requests:
    memory: 32Mi
    cpu: 10m
  limits:
    memory: 64Mi
    cpu: 50m
```

### templates/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "alertmanager-discord-webhook.fullname" . }}
  labels:
    {{- include "alertmanager-discord-webhook.labels" . | nindent 4 }}
data:
  config.yaml: |
    namespaces:
      {{- toYaml .Values.config.namespaces | nindent 6 }}
    labels:
      {{- toYaml .Values.config.labels | nindent 6 }}
    alertnames:
      {{- toYaml .Values.config.alertnames | nindent 6 }}
    excludeAlertnames:
      {{- toYaml .Values.config.excludeAlertnames | nindent 6 }}
    minSeverity: {{ .Values.config.minSeverity | quote }}
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "alertmanager-discord-webhook.fullname" . }}
  labels:
    {{- include "alertmanager-discord-webhook.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "alertmanager-discord-webhook.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "alertmanager-discord-webhook.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: DISCORD_WEBHOOK_URL
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.existingSecret | default (include "alertmanager-discord-webhook.fullname" .) }}
                  key: discord-webhook-url
            - name: CONFIG_PATH
              value: /etc/config/config.yaml
          volumeMounts:
            - name: config
              mountPath: /etc/config
              readOnly: true
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      volumes:
        - name: config
          configMap:
            name: {{ include "alertmanager-discord-webhook.fullname" . }}
```

## Step 5: ArgoCD Application

### homelab-gitops/infra-apps/alertmanager-discord-webhook.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: alertmanager-discord-webhook
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
    path: charts/alertmanager-discord-webhook
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
      - ServerSideApply=true
```

## Step 6: Update Alertmanager Config

Add the webhook receiver to `homelab-infra-charts/charts/kube-prometheus-stack/values/prod.yaml`:

```yaml
  alertmanager:
    enabled: true
    config:
      global:
        resolve_timeout: 5m
      route:
        receiver: discord
        group_by: ['alertname', 'namespace']
        group_wait: 30s
        group_interval: 5m
        repeat_interval: 4h
        routes:
          - receiver: discord
            match:
              severity: critical
            group_wait: 10s
            repeat_interval: 1h
      receivers:
        - name: discord
          webhook_configs:
            - url: 'http://alertmanager-discord-webhook.monitoring.svc:8080/webhook'
              send_resolved: true
    alertmanagerSpec:
      replicas: 1
      retention: 120h
```

## Step 7: Deploy

```bash
# Commit and push both repos
cd homelab-infra-charts && git add . && git commit -m "feat: add alertmanager discord webhook" && git push
cd homelab-gitops && git add . && git commit -m "feat: add alertmanager discord webhook app" && git push

# Sync ArgoCD
argocd app sync root
```

## Configuration

Edit `values/prod.yaml` under `config:` to filter which alerts are sent to Discord.

### Options

| Option | Type | Description |
|--------|------|-------------|
| `namespaces` | `[]string` | Only send alerts from these namespaces (empty = all) |
| `labels` | `map[string]string` | Only send alerts matching these labels (AND logic) |
| `alertnames` | `[]string` | Only send alerts with these names (empty = all) |
| `excludeAlertnames` | `[]string` | Exclude alerts with these names |
| `minSeverity` | `string` | Minimum severity: `info`, `warning`, `critical` |

### Examples

**Only critical alerts:**
```yaml
config:
  minSeverity: critical
```

**Only production namespace:**
```yaml
config:
  namespaces: ["production"]
```

**Specific alert names only:**
```yaml
config:
  alertnames: ["NodeDown", "PodCrashLooping"]
```

**Exclude watchdog alerts:**
```yaml
config:
  excludeAlertnames: ["Watchdog"]
```

**Multiple filters (AND logic):**
```yaml
config:
  namespaces: ["production"]
  labels:
    severity: critical
  minSeverity: warning
```

After changing values, sync ArgoCD:
```bash
argocd app sync alertmanager-discord-webhook
```

## Verify

```bash
# Check webhook service is running
kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager-discord-webhook

# Check ConfigMap was created
kubectl get configmap -n monitoring alertmanager-discord-webhook -o yaml

# Check logs for loaded config
kubectl logs -n monitoring -l app.kubernetes.io/name=alertmanager-discord-webhook

# Send a test alert
kubectl port-forward -n monitoring svc/alertmanager-operated 9093:9093 &
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TestAlert","severity":"critical","namespace":"production"},"annotations":{"summary":"Test alert"}}]'
```

## Discord Message Format

Each alert is sent as a Discord embed:

- **Title**: `[FIRING] AlertName` or `[RESOLVED] AlertName`
- **Description**: Alert description or summary annotation
- **Color**: Red (`0xFF0000`) for firing, Green (`0x00FF00`) for resolved
- **Fields**: All labels and annotations as name/value pairs
- **Footer**: Alert fingerprint
- **Timestamp**: Alert start time

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Webhook pod not starting | Check `kubectl logs -n monitoring <pod>` for errors |
| Discord not receiving alerts | Verify secret exists: `kubectl get secret -n monitoring alertmanager-discord-webhook` |
| Alerts not routing | Check Alertmanager UI at `alertmanager.uttutu.xyz` |
| Wrong alerts being sent | Check ConfigMap: `kubectl get configmap -n monitoring alertmanager-discord-webhook -o yaml` |
| Config changes not applied | Run `argocd app sync alertmanager-discord-webhook` |

## Security Notes

- The Discord webhook URL is stored as a Kubernetes Secret, NOT in Git
- The secret must be created manually via `kubectl create secret`
- Never commit the webhook URL to a public repository
