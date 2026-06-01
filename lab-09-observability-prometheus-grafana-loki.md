# Lab 9: Observability Stack — Prometheus, Grafana & Loki via Helm

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟠 Advanced
> **Estimated Time:** 120–150 minutes
> **Prerequisite:** Labs 1–8 completed

---

## 🎼 ELI5 — What Are We Actually Doing?

A great orchestra has more than just musicians. It has a **recording engineer** capturing every note, a **stage manager** noting every cue, and a **monitor in the booth** watching levels in real time. If something goes wrong — a microphone fails, a section drifts off-tempo, the audience's reaction sours — the team catches it immediately because *they're watching the right signals*.

Your Kubernetes cluster, until now, has been an orchestra performing in the dark. Pods are running. Maybe they're healthy, maybe they're not. Maybe one is crashing every 5 minutes and you don't know. Maybe response times tripled last Tuesday and nobody noticed. **You can't fix what you can't see.**

In this lab, you'll install the three pillars of observability:

1. **Prometheus** = the recording engineer for **metrics**. It scrapes numerical time-series data (request count, CPU usage, queue depth) from every Pod and stores it. Answers *"what is happening, in numbers?"*

2. **Grafana** = the booth monitor with the wall of screens. It queries Prometheus (and Loki) and turns the raw data into dashboards. Answers *"what does it look like right now and over time?"*

3. **Loki** = the stage manager for **logs**. Every line your apps print goes here, with labels for filtering. Answers *"what did the app actually say when this happened?"*

Together, they form the foundation of every modern Kubernetes platform. By the end of this lab, you'll have dashboards showing your cluster's health, you'll search logs across all Pods, and you'll have set up an alert that fires when something goes wrong.

---

## 🎯 Learning Objectives

- Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager + exporters) via Helm
- Understand Prometheus's pull-based scraping model
- Use ServiceMonitor / PodMonitor to scrape your own apps
- Write basic PromQL queries
- Install Loki + Promtail for log aggregation
- Configure Grafana to query both Prometheus and Loki
- Build a custom Grafana dashboard
- Create a PrometheusRule that fires alerts

---

## 📋 Prerequisites

| Item | Required | Check |
|------|----------|-------|
| AKS cluster | Yes | `kubectl get nodes` |
| Helm 3 | Yes | `helm version` |
| At least 4 vCPU and 8 GB RAM total | Yes | The stack is heavyweight |
| Persistent storage class | Yes | `kubectl get sc` |

> 💡 **Resource warning:** kube-prometheus-stack and Loki together need real horsepower. If your Lab 1 cluster was 2x `Standard_B2s`, scale up first or expect Pods to remain `Pending`.
>
> ```bash
> az aks scale -g rg-aks-lab07 -n aks-lab07 --node-count 3
> # Or change to Standard_B4ms (4 vCPU, 16 GB)
> ```

---

## 🧠 Key Concepts

**Metric** — A numerical measurement at a point in time. *"At 14:32:05, container X used 0.42 CPU cores."* Metrics are cheap, structured, and aggregatable.

**Log** — A timestamped text line. *"At 14:32:05, login failed for user xyz."* Logs are richer than metrics but expensive to store and search.

**Trace** — A record of a single request flowing through multiple services. (Not covered in this lab — see Tempo/Jaeger/OpenTelemetry as a follow-up.)

**Prometheus** — Pull-based metrics collector. It scrapes HTTP `/metrics` endpoints on a schedule. Most cluster components and many apps expose metrics in Prometheus format.

**ServiceMonitor / PodMonitor** — CRDs that tell the Prometheus Operator *"go scrape these targets."* You write small YAMLs instead of editing a giant Prometheus config.

**Grafana** — Open-source dashboard tool. Queries any data source (Prometheus, Loki, MySQL, CloudWatch, etc.).

**Loki** — Logs system from Grafana Labs. Stores log streams indexed only by labels (not full text), making it cheap.

**Promtail** (or Grafana Alloy) — Agent that runs on every node and ships container logs to Loki.

**Alertmanager** — Routes alerts. Prometheus evaluates rules and fires alerts; Alertmanager dedupes, groups, and sends them to Slack, email, PagerDuty, etc.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Install kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace observability
```

Create `kps-values.yaml` (sensible defaults for AKS):

```yaml
prometheus:
  prometheusSpec:
    retention: 7d
    resources:
      requests: { cpu: 200m, memory: 512Mi }
      limits:   { cpu: 1000m, memory: 2Gi }
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: managed-csi
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi
    # Discover all ServiceMonitors / PodMonitors regardless of label
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false

grafana:
  adminPassword: "ChangeMePlease!"
  persistence:
    enabled: true
    storageClassName: managed-csi
    size: 5Gi
  service:
    type: ClusterIP        # we'll port-forward; in prod use Ingress
  defaultDashboardsTimezone: Europe/Lisbon

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: managed-csi
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi
```

Install:

```bash
helm install kps prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --values kps-values.yaml \
  --version 65.0.0

# Wait for everything to come up (5-10 minutes)
kubectl get pods -n observability -w
```

When all pods show `Running`, press Ctrl+C.

### Step 2 — Access Grafana

```bash
kubectl port-forward -n observability svc/kps-grafana 3000:80
```

Open http://localhost:3000 — login as `admin` / `ChangeMePlease!`.

You'll see **dozens of preinstalled dashboards** under Dashboards → General. Try:
- **Kubernetes / Compute Resources / Cluster** → cluster-wide view
- **Kubernetes / Compute Resources / Namespace (Pods)** → drill into a namespace
- **Kubernetes / Compute Resources / Pod** → individual Pod details
- **Node Exporter / Nodes** → host-level metrics

The kube-prometheus-stack chart is one of the highest-value Helm installs in the entire ecosystem — you get a production-grade monitoring stack out of the box.

### Step 3 — Explore Prometheus directly

Port-forward Prometheus's UI:

```bash
kubectl port-forward -n observability svc/kps-kube-prometheus-stack-prometheus 9090:9090
```

Open http://localhost:9090. Try a few PromQL queries in the Graph tab:

- `up` — every scrape target's health (1 = up, 0 = down)
- `rate(node_cpu_seconds_total[5m])` — per-CPU usage rate
- `sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)` — CPU usage by namespace
- `kube_pod_status_phase{phase="Running"}` — count of Pods by phase

> 🧠 **PromQL is its own skill.** The basic operations: `rate()` for counters, `sum() by ()` for aggregation, `histogram_quantile()` for percentiles. Don't try to learn it all at once — pick it up via the dashboards you'll build.

### Step 4 — Deploy a sample app and scrape it with a ServiceMonitor

Let's deploy an app that exposes Prometheus metrics, then tell Prometheus to scrape it.

```bash
kubectl create namespace lab09
```

Create `sample-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: lab09
  labels: { app: sample-app }
spec:
  replicas: 2
  selector:
    matchLabels: { app: sample-app }
  template:
    metadata:
      labels: { app: sample-app }
    spec:
      containers:
        - name: app
          image: prom/prometheus:v2.55.0
          # prometheus itself exposes /metrics on 9090 — perfect demo target
          ports:
            - name: metrics
              containerPort: 9090
          resources:
            requests: { cpu: 50m, memory: 128Mi }
            limits:   { cpu: 200m, memory: 256Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: lab09
  labels: { app: sample-app }
spec:
  selector: { app: sample-app }
  ports:
    - name: metrics
      port: 9090
      targetPort: 9090
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app
  namespace: lab09
  labels:
    release: kps      # match the Helm release name so kps's Prometheus picks it up
spec:
  selector:
    matchLabels:
      app: sample-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

```bash
kubectl apply -f sample-app.yaml
```

> 🧠 **The `release: kps` label is critical.** It tells the Prometheus Operator that this ServiceMonitor belongs to your kps install. Without it, Prometheus ignores the ServiceMonitor.

In the Prometheus UI (Status → Targets), you should now see `lab09/sample-app` listed and `UP`.

In Grafana → Explore, select Prometheus, and query:

```
prometheus_http_requests_total{namespace="lab09"}
```

You're now scraping a custom app. **The same pattern applies to your real applications**: expose `/metrics` (most modern frameworks have a Prometheus client library), create a ServiceMonitor, you're done.

### Step 5 — Install Loki + Promtail for log aggregation

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Loki, single-binary mode, easier for learning
helm install loki grafana/loki-stack \
  --namespace observability \
  --set loki.persistence.enabled=true \
  --set loki.persistence.storageClassName=managed-csi \
  --set loki.persistence.size=10Gi \
  --set promtail.enabled=true

kubectl get pods -n observability -l app=loki

```

> 💡 **In production**, prefer the standalone `grafana/loki` chart with object storage (Azure Blob) instead of `loki-stack`. For learning, `loki-stack` is much simpler.

### Step 6 — Add Loki as a Grafana data source

In Grafana (still port-forwarded on :3000):

1. Configuration (gear icon) → **Data sources** → **Add data source**
2. Select **Loki**
3. URL: `http://loki.observability.svc.cluster.local:3100`
4. Click **Save & test** — should say "Data source successfully connected"

Now go to **Explore**, select **Loki** at the top, and try:

```
{namespace="lab09"}
```

You'll see logs from your sample-app Pods. Other useful queries:

```
{namespace="kube-system", pod=~"coredns.*"}
{namespace="observability"} |= "error"
{namespace="lab09"} | json | level="error"
```

LogQL is similar to PromQL but for logs. The pipe `|=` is a substring match; `|~` is regex; `| json` parses JSON logs into queryable fields.

### Step 7 — Build a custom Grafana dashboard

Create a dashboard that combines metrics and logs for `lab09`:

1. Dashboards → **New** → **New dashboard** → **Add visualization**
2. Data source: **Prometheus**
3. Query: `sum by (pod) (rate(prometheus_http_requests_total{namespace="lab09"}[5m]))`
4. Panel title: "HTTP Requests by Pod"
5. Apply

Add another panel for logs:

1. **Add → Visualization**
2. Data source: **Loki**
3. Query: `{namespace="lab09"}`
4. Visualization type: **Logs**
5. Apply

Click **Save** (top right), give the dashboard a name, save it.

You now have a single pane of glass for metrics and logs from your namespace. **In production, every team has dashboards like this — usually one per service — pinned and shared.**

### Step 8 — Create an alert rule

Let's alert when any Pod in lab09 has been restarting frequently.

Create `alert-rule.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: lab09-alerts
  namespace: observability
  labels:
    release: kps      # again, this label matters
spec:
  groups:
    - name: lab09.rules
      interval: 30s
      rules:
        - alert: PodRestartingFrequently
          expr: |
            increase(kube_pod_container_status_restarts_total{namespace="lab09"}[15m]) > 3
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} restarting frequently"
            description: "Pod {{ $labels.pod }} in {{ $labels.namespace }} has restarted >3 times in the last 15 minutes."
        - alert: HighMemoryUsage
          expr: |
            (sum by (pod, namespace) (container_memory_working_set_bytes{namespace="lab09"})
             / sum by (pod, namespace) (kube_pod_container_resource_limits{namespace="lab09",resource="memory"})) > 0.9
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} memory >90% of limit"
            description: "Pod {{ $labels.pod }} in {{ $labels.namespace }} has been above 90% memory usage for 10 minutes."
```

```bash
kubectl apply -f alert-rule.yaml
```

In the Prometheus UI → Alerts, you'll see your two new alerts. They'll be in the `Inactive` state (good — nothing is wrong yet).

### Step 9 — Trigger the alert (and connect it to Slack, optionally)

Force a Pod to crash repeatedly:

```yaml
# crashing-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crasher
  namespace: lab09
spec:
  replicas: 1
  selector: { matchLabels: { app: crasher } }
  template:
    metadata:
      labels: { app: crasher }
    spec:
      containers:
        - name: c
          image: busybox:1.36
          command: ["sh", "-c", "echo starting; sleep 5; exit 1"]
```

```bash
kubectl apply -f crasher.yaml

# Wait ~10 minutes, then check
kubectl get pod -n lab09
# CrashLoopBackOff with high restart count
```

In Prometheus → Alerts, `PodRestartingFrequently` should transition to `Firing` after enough crashes occur in the time window.

**To route alerts to Slack**, configure Alertmanager:

```yaml
# Add to your kps-values.yaml under alertmanager.config:
alertmanager:
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/services/T00/B00/XXXX'
    route:
      receiver: 'slack-default'
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
    receivers:
      - name: 'slack-default'
        slack_configs:
          - channel: '#alerts'
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}'
```

Then `helm upgrade kps prometheus-community/kube-prometheus-stack -n observability -f kps-values.yaml --version 65.0.0`.

> 💡 **Don't put real webhook URLs in version control.** Use a Secret + `secretRef` in production.

### Step 10 — Cleanup the crasher

```bash
kubectl delete deployment crasher -n lab09
```

---

## ✅ Validation Checklist

```bash
kubectl get pods -n observability                       # All Running
kubectl get servicemonitor -n lab09                     # sample-app present
# In Prometheus UI → Targets: lab09/sample-app shows "UP"
# In Grafana → Explore: queries against Prometheus AND Loki return data
# In Prometheus UI → Alerts: 2 lab09.rules alerts visible
```

---

## 🧹 Cleanup

```bash
helm uninstall kps -n observability
helm uninstall loki -n observability
kubectl delete namespace observability lab09

# Verify the underlying PVs are released or deleted
kubectl get pv
```
### Note
Should you experience any issue during the lab, I would recommend you spend some time troubleshooting as this will further help your learning.

---

## 🎓 What You Actually Learned

- **The three pillars of observability** are metrics, logs, and traces. This lab gave you the first two; tracing (Tempo/Jaeger/OpenTelemetry) is the natural follow-up.
- **Prometheus is pull-based.** Apps expose `/metrics`; Prometheus scrapes on a schedule. This model scales better than push-based and gives you health-checking for free (a missing scrape = a problem).
- **The Prometheus Operator changed the game.** Before it, Prometheus configs were a giant YAML you edited by hand. Now, ServiceMonitors and PodMonitors let teams self-onboard their apps with small declarative manifests.
- **kube-prometheus-stack is one of the highest-ROI Helm installs.** Out of the box, you get cluster-grade monitoring, sensible alerts, and ~30 dashboards. In production, you'd customize values.yaml extensively, but the starting point is excellent.
- **Loki indexes only labels, not log content.** This makes it dramatically cheaper than Elasticsearch for log volume — but means full-text search is slower (it streams matching log streams and filters in-flight).
- **Grafana is the conductor's wall of monitors.** It's where engineers actually live. Investing time in good dashboards pays off many times over during incidents.
- **Alerts must be actionable.** A noisy alert ignored at 3 AM is worse than no alert. Tune `for:` durations and thresholds carefully — the kube-prometheus-stack defaults are a good starting point.
- **Storage matters.** Both Prometheus and Loki need persistent storage. Sizing it correctly is its own discipline; for production, plan for retention × ingestion-rate.

---

## 🤔 Reflection Questions

1. Why is Prometheus pull-based instead of push-based? What problem does that solve?
2. Why does the ServiceMonitor need a label that matches the Prometheus install (`release: kps`)?
3. What's the difference between a ServiceMonitor and a PodMonitor? When would you use each?
4. Loki indexes only labels. What's the implication for queries vs Elasticsearch?
5. The `for: 5m` clause in an alert rule means what? Why not fire instantly?
6. If you have 100 microservices, would you create one dashboard per service or one giant dashboard? Why?

---

## 📚 Further Reading

- [Prometheus Operator Documentation](https://prometheus-operator.dev/docs/getting-started/introduction/)
- [PromQL Cheatsheet](https://promlabs.com/promql-cheat-sheet/)
- [Loki LogQL Documentation](https://grafana.com/docs/loki/latest/query/)
- [Awesome Prometheus Alerts (community-curated alert rules)](https://samber.github.io/awesome-prometheus-alerts/)
- [SRE: Monitoring Distributed Systems (Google SRE book chapter)](https://sre.google/sre-book/monitoring-distributed-systems/)

---

## ➡️ What's Next

You've built a cluster that's reliable, secure, scalable, and observable. The final lab brings it all together with the **modern enterprise delivery model**. In **Lab 10**, you'll install **ArgoCD** and run everything via **GitOps** — meaning every change to the cluster comes from a Git commit, not a `kubectl apply`. You'll set up a multi-tenant pattern with separate teams, scoped permissions, and automatic deployment from Git. By the end, you'll have built — at miniature scale — the kind of platform that real platform engineering teams operate at companies of every size.

---

*Author: Prince Chime · Series: Kubernetes on AKS*
