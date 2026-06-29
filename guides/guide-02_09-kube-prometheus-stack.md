# 02-07: Deploy kube-prometheus-stack via ArgoCD

## Prerequisites

- [ ] Kubernetes cluster running and healthy.
- [ ] ArgoCD bootstrapped and running (Guide 2)
- [ ] Root app synced and healthy
- [ ] MetalLB deployed
- [ ] Traefik + cert-manager deployed

## What We're Building

Full monitoring stack for the Kubernetes cluster:

| Component | Upstream Version | Purpose |
|-----------|-----------------|---------|
| Prometheus Operator | v0.92.0 | Manages Prometheus/Alertmanager instances |
| Prometheus | v2.55.1 | Metrics collection and storage |
| Alertmanager | v0.33.0 | Alert routing and notification |
| Grafana | latest | Visualization dashboards |
| kube-state-metrics | latest | Kubernetes object metrics |
| node-exporter | latest | Node-level metrics |

## kube-prometheus-stack Umbrella Chart

> [!NOTE]
> I have already created the chart in my [homelab-infra-charts repo](https://github.com/utkarsh-homelab/homelab-infra-charts), So you just need to clone the repo 😇
>
> `git clone git@github.com:utkarsh-homelab/homelab-infra-charts.git`

### charts/kube-prometheus-stack/Chart.yaml

```yaml
apiVersion: v2
name: kube-prometheus-stack
description: Vendored kube-prometheus-stack Helm chart for uttutu homelab
type: application
version: 1.0.0
appVersion: v0.92.0
dependencies:
  - name: kube-prometheus-stack
    version: 87.1.0
    repository: https://prometheus-community.github.io/helm-charts
```

### charts/kube-prometheus-stack/values/prod.yaml

```yaml
kube-prometheus-stack:
  defaultRules:
    create: true
    rules:
      alertmanager: true
      etcd: true
      general: true
      kubeApiserver: true
      kubeControllerManager: true
      kubePrometheusGeneral: true
      kubePrometheusNodeRecording: true
      kubeScheduler: true
      kubeStateMetrics: true
      kubernetesApps: true
      kubernetesResources: true
      kubernetesStorage: true
      kubernetesSystem: true
      kubelet: true
      network: true
      node: true
      nodeExporter: true
      prometheus: true
      prometheusOperator: true

  alertmanager:
    enabled: true
    alertmanagerSpec:
      replicas: 1
      retention: 120h
      resources:
        requests:
          memory: 128Mi
          cpu: 50m
        limits:
          memory: 256Mi
          cpu: 100m
      storage:
        volumeClaimTemplate:
          spec:
            storageClassName: nfs-csi
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 10Gi
    ingress:
      enabled: true
      ingressClassName: traefik
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hosts:
        - alertmanager.uttutu.xyz
      paths:
        - /
      tls:
        - secretName: wildcard-uttutu-tls
          hosts:
            - alertmanager.uttutu.xyz

  grafana:
    enabled: true
    adminUser: admin
    adminPassword: prom-operator
    ingress:
      enabled: true
      ingressClassName: traefik
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hosts:
        - grafana.uttutu.xyz
      path: /
      tls:
        - secretName: wildcard-uttutu-tls
          hosts:
            - grafana.uttutu.xyz
    persistence:
      enabled: true
      type: sts
      storageClassName: nfs-csi
      accessModes:
        - ReadWriteOnce
      size: 10Gi
    service:
      portName: http-web
    grafana.ini:
      server:
        root_url: https://grafana.uttutu.xyz
      auth:
        disable_login_form: false
      auth.anonymous:
        enabled: false
    sidecar:
      dashboards:
        enabled: true
        label: grafana_dashboard
        labelValue: "1"
        searchNamespace: ALL
      datasources:
        enabled: true
        defaultDatasourceEnabled: true
        isDefaultDatasource: true

  prometheus:
    prometheusSpec:
      replicas: 1
      retention: 15d
      resources:
        requests:
          memory: 512Mi
          cpu: 200m
        limits:
          memory: 1Gi
          cpu: 500m
      storageSpec:
        volumeClaimTemplate:
          spec:
            storageClassName: nfs-csi
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 50Gi
      podMonitorSelectorNilUsesHelmValues: false
      serviceMonitorSelectorNilUsesHelmValues: false
      serviceMonitorNamespaceSelector: {}
      podMonitorNamespaceSelector: {}
      ruleNamespaceSelector: {}
      additionalServiceMonitors:
        - name: argocd-metrics
          namespaceSelector:
            matchNames:
              - argocd
          selector:
            matchLabels:
              app.kubernetes.io/name: argocd-metrics
          endpoints:
            - port: metrics
              interval: 30s
        - name: argocd-server-metrics
          namespaceSelector:
            matchNames:
              - argocd
          selector:
            matchLabels:
              app.kubernetes.io/name: argocd-server-metrics
          endpoints:
            - port: metrics
              interval: 30s
        - name: argocd-repo-server-metrics
          namespaceSelector:
            matchNames:
              - argocd
          selector:
            matchLabels:
              app.kubernetes.io/name: argocd-repo-server-metrics
          endpoints:
            - port: metrics
              interval: 30s
    ingress:
      enabled: true
      ingressClassName: traefik
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hosts:
        - prometheus.uttutu.xyz
      paths:
        - /
      tls:
        - secretName: wildcard-uttutu-tls
          hosts:
            - prometheus.uttutu.xyz

  prometheusOperator:
    admissionWebhooks:
      enabled: false
    tls:
      enabled: false

  kubeStateMetrics:
    enabled: true

  nodeExporter:
    enabled: true

  windowsMonitoring:
    enabled: false
```

### Download sub-chart dependencies

```bash
cd ./homelab-infra-charts

# Add upstream Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Download sub-chart dependencies
helm dependency build charts/kube-prometheus-stack/
```

## kube-prometheus-stack GitOps App

> [!NOTE]
> The GitOps repo can be found [here](https://github.com/utkarsh-homelab/homelab-gitops)

The `apps/kube-prometheus-stack.yaml` references the infra-charts repo on sync wave 1:

> refer to the [gitops repo](https://github.com/utkarsh-homelab/homelab-gitops) for more details on sync strategy.

```yaml
# gitops/apps/kube-prometheus-stack.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
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
    path: charts/kube-prometheus-stack
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

## Verify Pi-hole DNS Records

No Cloudflare DNS records needed. Add these to Pi-hole (or verify they exist from [Previous Guide](./guide-02_08-traefik-cert-manager.md)):

| Domain | IP |
|--------|-----|
| `grafana.uttutu.xyz` | `192.168.0.200` |
| `prometheus.uttutu.xyz` | `192.168.0.200` |
| `alertmanager.uttutu.xyz` | `192.168.0.200` |

## Trigger ArgoCD Sync

```bash
argocd app sync root
```

## Verify

```bash
kubectl get pods -n monitoring
kubectl get prometheus -n monitoring
kubectl get servicemonitor -n monitoring | grep argocd
```

## Access Grafana

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80 &
```

Open `http://localhost:3000` — username `admin`, password `prom-operator`.

## Summary

| Component | Namespace | Ingress URL |
|-----------|-----------|-------------|
| Prometheus | monitoring | https://prometheus.uttutu.xyz |
| Alertmanager | monitoring | https://alertmanager.uttutu.xyz |
| Grafana | monitoring | https://grafana.uttutu.xyz |

## References

- [kube-prometheus-stack Helm Chart](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)
- [Prometheus Operator](https://prometheus-operator.dev/)