# 📘 Chapter 14 — Monitoring & Logging

> You can't manage what you can't measure. In production, **observability** is everything. This chapter covers how to see inside your cluster — metrics, logs, dashboards, and alerts — the complete stack used in real companies.

---

## 🔷 The 3 Pillars of Observability

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  METRICS   → Numbers over time                              │
│              CPU usage, memory, request rate, error rate    │
│              Tool: Prometheus + Grafana                     │
│                                                             │
│  LOGS      → Text records of events                         │
│              App logs, error messages, audit trails         │
│              Tool: ELK Stack / Loki + Grafana               │
│                                                             │
│  TRACES    → Request journey across services                │
│              How long each service took, where it failed    │
│              Tool: Jaeger / Zipkin / Tempo                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔷 Monitoring Stack Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     PRODUCTION CLUSTER                       │
│                                                              │
│  ┌──────────────┐    scrapes     ┌─────────────────┐         │
│  │  Prometheus  │ ◀──────────── │  Your App Pods  │         │
│  │  (collects   │                │  /metrics       │         │
│  │   metrics)   │ ◀──────────── │  Node Exporter  │         │
│  └──────┬───────┘                │  kube-state-    │         │
│         │                        │  metrics        │         │
│         │ queries                └─────────────────┘         │
│         ▼                                                    │
│  ┌──────────────┐    visualize   ┌─────────────────┐         │
│  │   Grafana    │ ──────────────▶│   Dashboards    │        │
│  │  (displays   │                │   Alerts        │         │
│  │   metrics)   │                │   Reports       │         │
│  └──────────────┘                └─────────────────┘         │
│                                                              │
│  ┌──────────────┐    collects    ┌─────────────────┐         │
│  │  Loki /      │ ◀──────────── │  All Pod Logs   │         │
│  │  ELK Stack   │                │  (via Promtail/ │         │
│  │  (logs)      │                │   Fluentd)      │         │
│  └──────────────┘                └─────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔷 PART 1 — Kubernetes Dashboard

### What is the K8s Dashboard?

A web UI to visualize your cluster — pods, deployments,
services, logs, resource usage — all in one place.

---

### 🛠️ Install Dashboard

```bash
# Install Kubernetes Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Verify pods running
kubectl get pods -n kubernetes-dashboard
# NAME                                         READY   STATUS
# dashboard-metrics-scraper-xxx                1/1     Running ✅
# kubernetes-dashboard-xxx                     1/1     Running ✅
```

---

### 🛠️ Create Admin Access for Dashboard

```yaml
# dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user-binding
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
roleRef:
  kind: ClusterRole
  name: cluster-admin              # full admin access
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f dashboard-admin.yaml

# Get login token
kubectl -n kubernetes-dashboard create token admin-user

# Output: eyJhbGciOiJSUzI1NiIsImtpZCI...  ← copy this

# Start proxy
kubectl proxy

# Open browser:
# http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

# Paste token → Login ✅
```

---

### 🔷 What Dashboard Shows You

```
Overview:
  → All namespaces at a glance
  → CPU and Memory usage per node
  → Pod status across cluster

Workloads:
  → Deployments, ReplicaSets, Pods
  → StatefulSets, DaemonSets, Jobs
  → Rolling update progress

Services:
  → All Services and their endpoints
  → Ingress rules

Storage:
  → PVs, PVCs and their binding status

Namespace view:
  → Switch between namespaces easily
  → Resource usage per namespace

Pod detail:
  → Live logs directly in browser
  → Resource usage graphs
  → Events and conditions
  → Exec into container from browser
```

---

## 🔷 PART 2 — Prometheus

### What is Prometheus?

```
Prometheus is a metrics DATABASE + scraping engine.

How it works:
  1. Your apps expose metrics at /metrics endpoint
  2. Prometheus scrapes (pulls) metrics on schedule
  3. Stores as time-series data
  4. You query with PromQL
  5. Grafana visualizes the data
```

---

### 🔷 What Prometheus Collects

```
Node metrics (via Node Exporter):
  → CPU usage per node
  → Memory usage per node
  → Disk I/O per node
  → Network traffic per node

K8s metrics (via kube-state-metrics):
  → Pod status (running/pending/failed)
  → Deployment replica counts
  → PVC status
  → ResourceQuota usage
  → Node conditions

App metrics (your own /metrics endpoint):
  → HTTP request rate
  → Error rate
  → Response time (p50, p95, p99)
  → Business metrics (orders/sec, signups/min)
  → Queue depth
```

---

### 🛠️ Install Prometheus Stack (Recommended Way)

```bash
# Add helm repo
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
# This installs: Prometheus + Grafana + AlertManager
# + Node Exporter + kube-state-metrics — everything at once
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set grafana.adminPassword=admin123

# Verify everything running
kubectl get pods -n monitoring
# NAME                                          READY   STATUS
# alertmanager-monitoring-xxx                   2/2     Running ✅
# monitoring-grafana-xxx                        3/3     Running ✅
# monitoring-kube-state-metrics-xxx             1/1     Running ✅
# monitoring-prometheus-node-exporter-xxx       1/1     Running ✅
# prometheus-monitoring-prometheus-xxx          2/2     Running ✅
```

---

### 🔷 Access Prometheus UI

```bash
# Port forward Prometheus
kubectl port-forward svc/monitoring-kube-prometheus-prometheus \
  9090:9090 -n monitoring

# Open: http://localhost:9090
```

---

### 🔷 PromQL — Prometheus Query Language

This is what you use to query metrics:

```promql
# Current CPU usage per pod
rate(container_cpu_usage_seconds_total{
  namespace="dev"
}[5m])

# Memory usage in MB per pod
container_memory_usage_bytes{
  namespace="production"
} / 1024 / 1024

# HTTP error rate (5xx errors)
rate(http_requests_total{status=~"5.."}[5m])
  /
rate(http_requests_total[5m])

# Pod restart count (detects crash loops)
increase(kube_pod_container_status_restarts_total[1h])

# Nodes CPU utilization %
100 - (avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
) * 100)

# Deployments with less replicas than desired
kube_deployment_status_replicas_available
  
kube_deployment_spec_replicas

# Pending pods (can't schedule)
kube_pod_status_phase{phase="Pending"} == 1
```

---

### 🛠️ ServiceMonitor — Tell Prometheus to Scrape Your App

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: dev
  labels:
    release: monitoring           # must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: my-app                 # scrape services with this label
  endpoints:
    - port: http                  # service port name
      path: /metrics              # metrics endpoint path
      interval: 30s               # scrape every 30 seconds
  namespaceSelector:
    matchNames:
      - dev                       # only this namespace
```

```yaml
# Your app Service must have matching labels
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: dev
  labels:
    app: my-app                   # matches ServiceMonitor selector
spec:
  selector:
    app: my-app
  ports:
    - name: http                  # matches ServiceMonitor port name
      port: 80
      targetPort: 8000
```

---

### 🔷 Expose Metrics From Your App

```python
# Python (Django/Flask) — using prometheus_client
from prometheus_client import Counter, Histogram, generate_latest

REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['endpoint']
)

# In your view/middleware:
REQUEST_COUNT.labels(
    method='GET',
    endpoint='/api/users',
    status='200'
).inc()

# Expose metrics endpoint
@app.route('/metrics')
def metrics():
    return generate_latest()
```

```javascript
// Node.js — using prom-client
const client = require('prom-client')
const register = new client.Registry()

const httpRequests = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [register]
})

// In middleware:
httpRequests.labels(req.method, req.route.path, res.statusCode).inc()

// Expose endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType)
  res.end(await register.metrics())
})
```

---

## 🔷 PART 3 — Grafana

### What is Grafana?

```
Grafana = visualization layer on top of Prometheus

  Prometheus = stores & queries metrics
  Grafana    = beautiful dashboards & alerts
```

---

### 🛠️ Access Grafana

```bash
# Port forward Grafana
kubectl port-forward svc/monitoring-grafana \
  3000:80 -n monitoring

# Open: http://localhost:3000
# Username: admin
# Password: admin123 (set during helm install)
```

---

### 🔷 Pre-built Dashboards (Import These)

```
Grafana has thousands of community dashboards.
Import by ID directly in Grafana UI.

Essential dashboards to import:

  ID: 315    → Kubernetes cluster monitoring (by namespace)
  ID: 6417   → Kubernetes pod + container metrics
  ID: 1860   → Node Exporter full (hardware metrics)
  ID: 13332  → Kubernetes cluster overview
  ID: 12740  → Kubernetes deployment metrics
  ID: 11074  → Node + cluster resource usage

Import steps:
  Grafana → Dashboards → Import → Enter ID → Load ✅
```

---

### 🛠️ Create Custom Dashboard Panel

```
In Grafana UI:
  1. New Dashboard → Add Panel

  2. Data source: Prometheus

  3. Query (PromQL):
     sum(rate(http_requests_total{
       namespace="production"
     }[5m])) by (service)

  4. Visualization: Time series graph

  5. Title: "HTTP Request Rate by Service"

  6. Save ✅
```

---

### 🛠️ AlertManager — Set Up Alerts

```yaml
# prometheusrule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: app.rules
      rules:
        # Alert when pod keeps crashing
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total[15m]) > 3
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
            description: "Pod has restarted more than 3 times in 15 min"

        # Alert when deployment has fewer pods than desired
        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_status_replicas_available
              
            kube_deployment_spec_replicas
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Deployment {{ $labels.deployment }} is degraded"
            description: "Available replicas less than desired for 10 min"

        # Alert when node CPU is high
        - alert: NodeHighCPU
          expr: |
            100 - (avg by (instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            ) * 100) > 85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Node {{ $labels.instance }} CPU > 85%"

        # Alert when pod is pending too long
        - alert: PodStuckPending
          expr: |
            kube_pod_status_phase{phase="Pending"} == 1
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} stuck in Pending"
```

---

### 🔷 AlertManager Routes — Send to Slack/PagerDuty

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-monitoring-kube-prometheus-alertmanager
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'

    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'

      routes:
        - match:
            severity: critical
          receiver: pagerduty-critical    # wake someone up
          continue: true

        - match:
            severity: warning
          receiver: slack-notifications   # just notify

    receivers:
      - name: 'slack-notifications'
        slack_configs:
          - channel: '#k8s-alerts'
            title: '{{ .GroupLabels.alertname }}'
            text: >-
              {{ range .Alerts }}
                *Alert:* {{ .Annotations.summary }}
                *Details:* {{ .Annotations.description }}
              {{ end }}'

      - name: 'pagerduty-critical'
        pagerduty_configs:
          - routing_key: 'your-pagerduty-key'
```

---

## 🔷 PART 4 — Logging with Loki

### Loki vs ELK Stack

```
ELK Stack (Elasticsearch + Logstash + Kibana):
  → Full-featured, indexes everything
  → Powerful search
  → Expensive (high memory for Elasticsearch)
  → Complex to operate

Loki (by Grafana Labs):
  → Lightweight — doesn't index log content
  → Indexes only labels (namespace, pod, app)
  → Much cheaper to run
  → Integrates natively with Grafana
  → Perfect for k8s (uses same labels)

For most k8s setups: Loki is recommended ✅
```

---

### 🛠️ Install Loki Stack

```bash
# Add Grafana helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki + Promtail (log collector)
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.enabled=true \
  --set grafana.enabled=false    # already have Grafana installed

# Verify
kubectl get pods -n monitoring | grep loki
# loki-0                      1/1   Running ✅
# loki-promtail-xxxxx          1/1   Running ✅  (one per node — DaemonSet)
```

---

### 🔷 How Loki Works

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Every node has a Promtail pod (DaemonSet)              │
│       ↓                                                 │
│  Promtail reads /var/log/pods/* on each node            │
│       ↓                                                 │
│  Tags logs with labels: namespace, pod, container       │
│       ↓                                                 │
│  Ships to Loki (central log store)                      │
│       ↓                                                 │
│  Query logs from Grafana using LogQL                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### 🔷 Add Loki as Data Source in Grafana

```
Grafana → Configuration → Data Sources → Add
  Type: Loki
  URL:  http://loki:3100
  Save & Test ✅
```

---

### 🔷 LogQL — Query Your Logs

```logql
# All logs from a namespace
{namespace="production"}

# Logs from specific app
{namespace="production", app="django-notes"}

# Filter for errors
{namespace="production"} |= "ERROR"

# Regex filter
{namespace="production"} |~ "Exception|Error|FATAL"

# Exclude noisy logs
{namespace="production"} != "health check"

# Count error rate over time
rate({namespace="production"} |= "ERROR" [5m])

# Parse and extract fields
{namespace="production"}
  | json
  | level="error"
  | line_format "{{.message}}"

# Logs from last 30 minutes with pod name
{namespace="production", pod=~"django-.*"}
  |= "ERROR"
  [30m]
```

---

### 🛠️ Grafana Explore — Query Logs Live

```
In Grafana:
  1. Left menu → Explore
  2. Data source: Loki
  3. Enter LogQL query:
     {namespace="production"} |= "ERROR"
  4. See real-time logs ✅
  5. Click log line → see full details
  6. Jump to metrics at same timestamp → correlate ✅
```

---

## 🔷 PART 5 — Full Observability Flow

### Real Incident Response Example

```
🚨 Alert fires: "HTTP error rate > 5% in production"
          │
          ▼
1. Grafana dashboard → see spike in 500 errors
   → affects /api/orders endpoint

          │
          ▼
2. Prometheus query:
   kube_pod_container_status_restarts_total
   → orders-service pods are restarting!

          │
          ▼
3. kubectl get pods -n production
   → orders-service-xxx   CrashLoopBackOff ❌

          │
          ▼
4. Loki logs query:
   {namespace="production", app="orders-service"}
   |= "ERROR"
   → "Database connection refused: mysql-service:3306"

          │
          ▼
5. kubectl get pods -n production | grep mysql
   → mysql-0   0/1   Pending ❌ (PVC stuck!)

          │
          ▼
6. kubectl describe pvc mysql-data-mysql-0
   → "no persistent volumes available"

          │
          ▼
7. Fix: provision storage, PVC binds, mysql starts,
   orders-service recovers, alerts clear ✅

Total time to identify root cause: 5 minutes
Without observability: hours of guessing 😅
```

---

## 🔷 Production Monitoring Checklist

```
Metrics to always monitor:
  ✅ Pod CPU and memory usage
  ✅ Pod restart count (CrashLoopBackOff detection)
  ✅ Pending pods (scheduling failures)
  ✅ Node CPU, memory, disk usage
  ✅ HTTP error rate (4xx, 5xx)
  ✅ HTTP latency (p95, p99)
  ✅ PVC usage (disk filling up)
  ✅ ResourceQuota usage

Alerts to always have:
  ✅ Pod crash looping
  ✅ Deployment replicas mismatch
  ✅ Node not ready
  ✅ High CPU/memory (>85%)
  ✅ PVC almost full (>80%)
  ✅ Certificate expiry (<30 days)
  ✅ Pending pods (>15 min)

Logs to always capture:
  ✅ Application error logs
  ✅ K8s audit logs (who did what)
  ✅ Node system logs
```

---

## ✅ Chapter 14 Summary

| Concept | Key Point |
|---|---|
| Metrics | Numbers over time — Prometheus collects them |
| Logs | Text events — Loki/ELK collects them |
| Prometheus | Scrapes /metrics endpoints, stores time-series data |
| PromQL | Query language for Prometheus metrics |
| Grafana | Visualization — dashboards + alerts on top of Prometheus |
| ServiceMonitor | Tells Prometheus which services to scrape |
| AlertManager | Routes alerts to Slack, PagerDuty, email |
| Loki | Lightweight log aggregation for k8s |
| LogQL | Query language for Loki logs |
| Promtail | DaemonSet that ships pod logs to Loki |
| Node Exporter | Collects hardware metrics from each node |
| kube-state-metrics | Collects k8s object state metrics |

---

## 🧠 Interview Questions You Can Now Nail

1. **"How do you monitor a Kubernetes cluster in production?"**
   → Full observability stack: Prometheus for metrics (scrapes /metrics endpoints, node exporter for hardware, kube-state-metrics for k8s objects), Grafana for dashboards and alerts via AlertManager, Loki for log aggregation via Promtail DaemonSet.

2. **"What is the difference between Prometheus and Grafana?"**
   → Prometheus is a metrics database and scraping engine — it collects and stores time-series data and you query it with PromQL. Grafana is a visualization layer — it queries Prometheus and renders dashboards, charts, and alerts.

3. **"How does Prometheus discover what to scrape in k8s?"**
   → Via ServiceMonitor custom resources (part of kube-prometheus-stack). ServiceMonitor selects Services by labels and defines the scrape endpoint and interval. Prometheus watches for ServiceMonitor objects and auto-configures scrape targets.

4. **"What is Loki and how is it different from ELK?"**
   → Both aggregate logs but Loki only indexes labels (namespace, pod, app) not log content — making it much cheaper and simpler to run. ELK indexes everything for powerful full-text search but is resource-heavy. Loki integrates natively with Grafana.

5. **"How would you debug a production incident in k8s?"**
   → Check Grafana alerts for what's failing, use Prometheus to identify which pods/services are affected, check pod status with kubectl, query Loki logs for the error message, trace root cause from the error. Full observability stack makes this a 5-minute exercise instead of hours.

---

Ready for **Chapter 15 → Helm & CRDs & Operators** — the tools that make deploying complex apps (like databases, Kafka, cert-manager) into k8s a one-liner? The final pieces before the projects. Say **"next"** 🚀
