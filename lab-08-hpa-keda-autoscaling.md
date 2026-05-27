# Lab 8: Horizontal Pod Autoscaling & KEDA

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟠 Advanced
> **Estimated Time:** 90–120 minutes
> **Prerequisite:** Labs 1–7 completed

---

## 🎼 ELI5 — What Are We Actually Doing?

A great orchestra adapts to the audience. A small jazz club needs three musicians. A festival main stage needs forty. The same music, scaled to the venue.

Up to now, your applications have had a fixed number of Pods. You decided "3 replicas" or "4 replicas" at deploy time, and that's what you got — whether ten people were using your app or ten thousand. That's wasteful when traffic is low (paying for idle Pods) and dangerous when traffic spikes (Pods overloaded, users getting errors).

The fix is **autoscaling** — letting Kubernetes add and remove musicians based on what the audience needs.

There are two flavors:

1. **Horizontal Pod Autoscaler (HPA)** — Built into Kubernetes. Scales based on CPU or memory. Works great for _web apps_ where load shows up as CPU usage. _"Average CPU above 70%? Add more musicians. Below 30%? Send some home."_

2. **KEDA (Kubernetes Event-Driven Autoscaler)** — A CNCF project. Scales on dozens of richer signals: queue depth in Azure Service Bus, message count in RabbitMQ, custom Prometheus metrics, Kafka lag, even cron schedules. Critical for _workers_ and _event-driven apps_ where CPU isn't the right signal. _"50,000 messages waiting in the queue? Scale to 30 workers. Empty queue? Scale to zero."_

By the end of this lab, your apps will scale up automatically when busy, scale down when idle, and even scale to zero when there's literally no work — saving real money on a real cluster.

---

## 🎯 Learning Objectives

- Install the Metrics Server (or verify AKS has it)
- Create an HPA that scales on CPU
- Generate load and watch the HPA react in real time
- Understand HPA's algorithm and tuning knobs (`stabilizationWindow`, behaviors)
- Install KEDA via Helm
- Create a `ScaledObject` that scales on a custom trigger
- Scale on Azure Service Bus queue depth (real enterprise pattern)
- Use cron-based scaling for predictable workloads
- Understand the relationship between HPA, KEDA, VPA, and Cluster Autoscaler

---

## 📋 Prerequisites

| Item                         | Required             | Check                             |
| ---------------------------- | -------------------- | --------------------------------- |
| AKS cluster                  | Yes                  | `kubectl get nodes`               |
| Helm 3                       | Yes                  | `helm version`                    |
| Metrics Server               | Yes (built into AKS) | `kubectl top nodes` should work   |
| (Optional) Azure Service Bus | Recommended          | For the enterprise demo in Step 7 |

---

## 🧠 Key Concepts

**Metrics Server** — A cluster add-on that collects CPU/memory metrics from nodes and Pods. AKS has it preinstalled. Other autoscalers depend on it.

**HPA (Horizontal Pod Autoscaler)** — A controller that adjusts a Deployment's `replicas` based on metrics. Built into Kubernetes.

**KEDA** — An add-on that extends HPA with 60+ event sources (queues, databases, custom metrics). Implemented as an HPA "metric server" — under the hood it actually creates a regular HPA.

**ScaledObject** — KEDA's main resource. References a target Deployment and a list of triggers.

**Scale to zero** — KEDA can scale a workload down to 0 Pods when idle (HPA can't go below 1). The first event then triggers immediate scale-up.

**Cluster Autoscaler** — _Different layer._ Adjusts the number of _nodes_ (VMs), not Pods. AKS has built-in support; we'll touch on it.

**VPA (Vertical Pod Autoscaler)** — Also different. Adjusts a Pod's CPU/memory requests/limits instead of the replica count. Less commonly used; not covered in depth here.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Verify Metrics Server works

```bash
kubectl top nodes
kubectl top pods --all-namespaces
```

If both work, Metrics Server is healthy. AKS includes it by default.

### Step 2 — Set up the lab namespace and a target app

```bash
kubectl create namespace lab08
kubectl config set-context --current --namespace=lab08
```

Create `app.yaml` — a Pod that consumes CPU on demand (a load-test target):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels: { app: php-apache }
  template:
    metadata:
      labels: { app: php-apache }
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "200m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector: { app: php-apache }
  ports:
    - port: 80
```

```bash
kubectl apply -f app.yaml
kubectl wait --for=condition=available deployment/php-apache --timeout=120s
```

> 🧠 **HPA needs `resources.requests.cpu` to be set.** It scales based on the percentage of _requested_ CPU being used. No request = no HPA.

### Step 3 — Create your first HPA

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100 # double the pods if needed
          periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50 # halve the pods over 60s, but cautiously
          periodSeconds: 60
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
```

### Step 4 — Generate load and watch the HPA react

In one terminal, watch:

```bash
kubectl get hpa php-apache -w
```

In another, generate continuous load:

```bash
kubectl run load-generator --rm -i --tty --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

After ~60 seconds, you'll see the HPA's `TARGETS` column climb above 50%, and replicas will start increasing — 1 → 2 → 4 → 6 → up to the max of 10.

**Stop the load** (Ctrl+C the load generator). Watch the scale-down:

```bash
kubectl get hpa php-apache -w
```

You'll see TARGETS drop back to 0%, and after the 5-minute stabilization window, replicas will gradually return to 1.

> 🧠 **Why scale-down is slower than scale-up.** Aggressive scale-down causes flapping — scale down, traffic returns, scale back up, traffic dies, scale down again. The 5-minute `stabilizationWindowSeconds` for `scaleDown` is the canonical default. Scale up fast (users are waiting), scale down slow.

### Step 5 — Scale on memory and multiple metrics

You can also scale on memory or multiple metrics simultaneously. The HPA picks the metric demanding the most replicas:

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

For most web apps, CPU is enough. Memory-based scaling is risky because most apps' memory grows but rarely shrinks — you can end up oscillating or never scaling down.

### Step 6 — Install KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

kubectl create namespace keda
helm install keda kedacore/keda --namespace keda

kubectl get pods -n keda
```

You should see three running pods: `keda-operator`, `keda-operator-metrics-apiserver`, and `keda-admission-webhooks`.

### Step 7 — Scale on Azure Service Bus queue depth (the enterprise pattern)

This is the canonical "scale workers based on work" use case.

> 💡 **Want to skip Azure setup?** Jump to Step 8 (cron-based scaling) which doesn't need any external service.

**Create a Service Bus namespace and queue:**

```bash
RG=rg-aks-lab07          # or your existing RG
LOC=westeurope
SB_NS=sblab08-${RANDOM}
QUEUE=tasks

az servicebus namespace create -g $RG -n $SB_NS -l $LOC --sku Standard
az servicebus queue create -g $RG --namespace-name $SB_NS -n $QUEUE

# Get a connection string
SB_CONN=$(az servicebus namespace authorization-rule keys list \
  -g $RG --namespace-name $SB_NS -n RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv)
```

**Store the connection string as a Secret:**

```bash
kubectl create secret generic sb-secret \
  --from-literal=connection="$SB_CONN" \
  -n lab08
```

**Deploy a worker app and a TriggerAuthentication + ScaledObject:**

```yaml
# worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue-worker
spec:
  replicas: 0
  selector: { matchLabels: { app: queue-worker } }
  template:
    metadata:
      labels: { app: queue-worker }
    spec:
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - "echo 'I am a worker; pretend to process'; sleep 60"
          resources:
            requests: { cpu: 50m, memory: 32Mi }
            limits: { cpu: 200m, memory: 64Mi }
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: sb-auth
spec:
  secretTargetRef:
    - parameter: connection
      name: sb-secret
      key: connection
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-worker-scaler
spec:
  scaleTargetRef:
    name: queue-worker
  minReplicaCount: 0 # scale to ZERO when no messages
  maxReplicaCount: 20
  pollingInterval: 15
  cooldownPeriod: 60
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: tasks
        messageCount: "5" # one replica per 5 messages
      authenticationRef:
        name: sb-auth
```

```bash
kubectl apply -f worker.yaml
kubectl get scaledobject
kubectl get hpa            # KEDA created an HPA under the hood
kubectl get pods           # 0 worker pods — scaled to zero!
```

**Now send messages and watch the magic:**

```bash
# Send 50 messages
for i in $(seq 1 50); do
  az servicebus queue message send \
    -g $RG --namespace-name $SB_NS --queue-name $QUEUE \
    --body "task-$i"
done

# Watch the worker scale up from zero
kubectl get pods -w
```

if the above fails, use the method below:

## Sending Messages to Azure Service Bus Queue (using SAS Token)

If the Azure CLI lacks a `az servicebus queue message send` command, use this bash script to send messages via the Service Bus REST API with a SAS token generated from your connection string.

**Prerequisites:**

- `jq` installed (for URL encoding). If not, see alternative below.
- `openssl` and `base64` (available on most systems).
- Your Service Bus namespace, queue name, and connection string variables: `$SB_NS`, `$QUEUE`, `$SB_CONN`.

### Script

```bash
# Ensure these variables are set in your terminal
# RG, SB_NS, QUEUE, SB_CONN

# Extract key and key name from connection string
KEY_NAME="RootManageSharedAccessKey"
KEY=$(echo "$SB_CONN" | sed -n 's/.*SharedAccessKey=\([^;]*\).*/\1/p')
RESOURCE_URI="https://${SB_NS}.servicebus.windows.net/${QUEUE}"

# Generate SAS token (valid 1 hour)
expiry=$(($(date +%s) + 3600))
string_to_sign="${RESOURCE_URI}\n${expiry}"
signature=$(printf "$string_to_sign" | openssl sha256 -hmac "$KEY" -binary | base64 | tr -d '\n')
sas_token="SharedAccessSignature sr=${RESOURCE_URI}&sig=$(echo -n "$signature" | jq -sRr @uri)&se=${expiry}&skn=${KEY_NAME}"

# Send 50 messages to the queue
for i in {1..50}; do
  curl -X POST \
    -H "Authorization: ${sas_token}" \
    -H "Content-Type: text/plain" \
    -d "task-${i}" \
    "${RESOURCE_URI}/messages"
  echo " Sent task-${i}"
done
```

Within ~15 seconds (one polling interval), KEDA will start spinning up workers. With 50 messages and 5 per replica, expect ~10 replicas. As workers consume messages and the queue drains, the count scales back down — eventually to **zero** when the queue is empty.

> 🧠 **This is huge for cost.** Worker fleets that previously sat idle 80% of the time now consume zero compute when there's no work. Your finance team will love you.

### Step 8 — Cron-based scaling (no external dependencies)

KEDA also supports time-based triggers. Useful for predictable patterns: scale up before business hours, scale down at night.

```yaml
# cron-scaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: business-hours-app
spec:
  replicas: 1
  selector: { matchLabels: { app: business-hours-app } }
  template:
    metadata:
      labels: { app: business-hours-app }
    spec:
      containers:
        - name: app
          image: nginx:1.27-alpine
          resources:
            requests: { cpu: 50m, memory: 64Mi }
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: business-hours-scaler
spec:
  scaleTargetRef:
    name: business-hours-app
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: cron
      metadata:
        timezone: Europe/Lisbon
        start: "0 8 * * 1-5" # 08:00 Mon–Fri
        end: "0 19 * * 1-5" # 19:00 Mon–Fri
        desiredReplicas: "5"
```

```bash
kubectl apply -f cron-scaler.yaml
kubectl get scaledobject
```

Between 8 AM and 7 PM weekdays, this Deployment runs 5 replicas. Outside that window, it drops to 1. **Predictable cost optimization for workloads with predictable load patterns.**

### Step 9 — Combine HPA-style metrics with KEDA

KEDA's strength is composition. You can have a ScaledObject with multiple triggers — CPU + queue depth + cron — and it scales to whichever is highest. This handles the _"scale on whatever signal makes sense right now"_ problem elegantly.

```yaml
triggers:
  - type: cpu
    metricType: Utilization
    metadata:
      value: "70"
  - type: azure-servicebus
    metadata:
      queueName: tasks
      messageCount: "5"
    authenticationRef:
      name: sb-auth
  - type: cron
    metadata:
      timezone: Europe/Lisbon
      start: "0 8 * * 1-5"
      end: "0 19 * * 1-5"
      desiredReplicas: "5"
```

### Step 10 — Cluster Autoscaler (briefly)

HPA and KEDA scale Pods. But what if there's no room on existing nodes for new Pods?

The **Cluster Autoscaler** adjusts the number of VMs in your AKS cluster. It's enabled at cluster level:

```bash
az aks update -g $RG -n aks-lab07 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 8
```

Now if Pods can't be scheduled (e.g., HPA wants 50 Pods but there's only room for 30), AKS adds nodes. When nodes become idle for ~10 minutes, it removes them.

**The full chain:** HPA/KEDA scales Pods → Pods are unschedulable → Cluster Autoscaler adds nodes → load drops → Pods scale down → nodes empty → nodes get removed.

---

## ✅ Validation Checklist

```bash
kubectl get hpa                          # php-apache HPA listed
kubectl get scaledobject                 # KEDA ScaledObjects listed
kubectl get hpa | grep keda              # KEDA-managed HPAs (KEDA creates these)
kubectl top pods                         # Real-time CPU usage visible
```

---

## 🧹 Cleanup

```bash
kubectl delete namespace lab08

# Remove KEDA if not needed
helm uninstall keda -n keda
kubectl delete namespace keda

# Remove Service Bus namespace
az servicebus namespace delete -g $RG -n $SB_NS

# Disable cluster autoscaler if you don't want it
az aks update -g $RG -n aks-lab07 --disable-cluster-autoscaler
```

---

## 🎓 What You Actually Learned

- **HPA scales horizontally based on metrics, not magic.** Set `resources.requests`, define a target utilization, and the controller does the rest.
- **`stabilizationWindowSeconds` is the most important tuning knob.** Aggressive scale-down causes flapping; the default (5 min) exists for good reasons.
- **HPA's biggest limit is the metric source.** CPU and memory are easy. Anything else needs a custom metrics adapter — or KEDA.
- **KEDA solves "scale on the right signal."** Queue depth, Kafka lag, Prometheus metrics, cron schedules — KEDA understands all of them and translates them into HPA mechanics.
- **Scale-to-zero is real and powerful.** For event-driven and worker workloads, idle compute genuinely costs money. KEDA + scale-to-zero can drop that to actual zero.
- **Multi-trigger ScaledObjects compose elegantly.** Pick the highest signal at any moment — peak CPU, queue depth spike, or business-hours guarantee.
- **Pod scaling and node scaling are different layers.** HPA/KEDA scale Pods. Cluster Autoscaler scales nodes. Both must be enabled to actually save money on bursty workloads.
- **AKS has cluster autoscaler built in.** Enable with one az CLI command. Combined with KEDA scale-to-zero, you can run worker queues that genuinely cost ~$0 when idle.

---

## 🤔 Reflection Questions

1. Why is scaling on memory utilization usually a bad idea?
2. Your HPA targets 70% CPU and currently has 4 Pods averaging 100% CPU. How many Pods will it ask for? (Hint: it's a ratio.)
3. Why can't built-in HPA scale to zero, but KEDA can?
4. KEDA creates an HPA under the hood. Why? Why not just bypass HPA entirely?
5. If you have a `cron` trigger setting `desiredReplicas: 5` and a `cpu` trigger asking for 8, what does KEDA do?
6. When would you choose VPA over HPA?

---

## 📚 Further Reading

- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [KEDA Documentation](https://keda.sh/)
- [KEDA Scalers Catalog](https://keda.sh/docs/latest/scalers/)
- [AKS Cluster Autoscaler](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler)

---

## ➡️ What's Next

Your apps now scale, are secure, and run reliably. But how do you know what they're actually _doing_? In **Lab 9**, you'll deploy a complete observability stack — **Prometheus** for metrics, **Grafana** for dashboards, and **Loki** for logs — all installed via Helm using the kube-prometheus-stack chart. You'll watch real-time data from your cluster, build dashboards, and set up alerts. This is the "I can sleep at night" lab.

---

_Author: Prince Chime · Series: Kubernetes on AKS_
