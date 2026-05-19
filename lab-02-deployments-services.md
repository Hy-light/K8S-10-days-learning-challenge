# Lab 2: Deployments, ReplicaSets & Services

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟢 Beginner
> **Estimated Time:** 60–75 minutes
> **Prerequisite:** Lab 1 completed (or an existing AKS cluster you can reach with `kubectl`)

---

## 🎼 ELI5 — What Are We Actually Doing?

In Lab 1, you cued a single musician onto the stage. It worked — but only as long as nothing went wrong. If that musician got sick, the music stopped. If the audience wanted louder sound, you had no way to add more players. And if someone in the audience asked *"where do I send my applause?"*, there was no fixed address — every new musician sits in a different chair.

A real orchestra solves these three problems with three roles:

1. **The personnel manager** keeps a roster: *"We must always have exactly 3 violinists on stage. If one goes home, hire another."* In Kubernetes, this is a **ReplicaSet**.
2. **The artistic director** plans changes over time: *"Next week, we're switching from Mozart to Beethoven. Swap the musicians out gradually so the audience never hears silence."* In Kubernetes, this is a **Deployment** — and it uses a ReplicaSet to do its work.
3. **The box office** gives the audience a single, stable phone number to reach the orchestra, no matter which specific musicians are playing tonight. In Kubernetes, this is a **Service**.

In this lab, you'll build all three. You'll deploy an app that **survives failure**, **scales on demand**, **upgrades without downtime**, and is **reachable at a stable address** — the four properties that separate "I ran a container" from "I built a real system."

---

## 🎯 Learning Objectives

By the end of this lab, you will:

- Understand the relationship between Deployments, ReplicaSets, and Pods
- Create a Deployment using YAML
- Scale your application horizontally
- Perform a rolling update and a rollback
- Expose your app internally with a `ClusterIP` Service
- Expose your app externally with a `LoadBalancer` Service (gets a real public IP from Azure)
- Understand label selectors — how Services find their Pods

---

## 📋 Prerequisites

| Tool | Required | Check Command |
|------|----------|---------------|
| AKS cluster from Lab 1 | Yes | `kubectl get nodes` |
| `kubectl` connected | Yes | `kubectl cluster-info` |

> 💡 If you deleted the cluster, recreate it with the Lab 1 commands. It takes ~10 minutes.

---

## 🧠 Key Concepts (Read Before You Start)

**Deployment** — A high-level object that manages your application over time. You tell it *"I want 3 copies of this app, using version 1.27"*, and it figures out how to get there and stay there. You almost never create Pods directly in production; you create Deployments.

**ReplicaSet** — The engine the Deployment uses. Its only job: *"Maintain N healthy copies of this Pod template."* If a Pod dies, the ReplicaSet creates a new one. You rarely interact with ReplicaSets directly — Deployments manage them for you.

**Service** — A stable network endpoint that load-balances requests to a set of Pods. Pods come and go; Services don't. They use **labels** to find which Pods to send traffic to.

**Service Types** (we'll use the first two today):
- `ClusterIP` — Reachable only from inside the cluster (the default). Like an internal phone extension.
- `LoadBalancer` — Reachable from the public internet. On AKS, this provisions a real Azure Load Balancer with a public IP.
- `NodePort` — Exposes a port on every node. Rarely used directly in production.

**Labels & Selectors** — Labels are sticky notes on Pods (`app: hello`, `env: prod`). Selectors are filters that match those notes (*"send traffic to anyone with the label `app: hello`"*). This is how Kubernetes wires loosely coupled things together.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Set up a working namespace

A namespace is a logical partition inside the cluster — like reserving a private rehearsal room within the concert hall.

```bash
kubectl create namespace lab02
kubectl config set-context --current --namespace=lab02
```

The second command sets `lab02` as the default namespace for this session, so you don't have to type `-n lab02` on every command.

### Step 2 — Write your first Deployment manifest

Create a file called `hello-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
  labels:
    app: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
```

**Reading this manifest in plain English:**

- `kind: Deployment` → "I want the artistic director to manage this."
- `replicas: 3` → "Always keep 3 musicians on stage."
- `selector.matchLabels` → "These are the musicians I'm responsible for (anyone wearing the `app: hello` badge)."
- `template` → "Here's the sheet music every new musician should follow."

> 🧠 **Crucial concept:** The labels in `selector.matchLabels` MUST match the labels in `template.metadata.labels`. This is how the Deployment knows which Pods belong to it. Mismatch them and you get bizarre behavior.

### Step 3 — Apply the Deployment

```bash
kubectl apply -f hello-deployment.yaml
```

**Watch the orchestra take the stage:**

```bash
kubectl get deployment hello-deploy
kubectl get replicaset
kubectl get pods --show-labels
```

Notice the chain: **Deployment → ReplicaSet → 3 Pods**. The ReplicaSet name is auto-generated (something like `hello-deploy-7c9f8b6d4`).

### Step 4 — Prove that Pods auto-heal

Let's kill a musician and watch the personnel manager hire a replacement:

```bash
# Get one Pod's name
POD_TO_KILL=$(kubectl get pods -l app=hello -o jsonpath='{.items[0].metadata.name}')
echo "Killing: $POD_TO_KILL"

# Delete it
kubectl delete pod $POD_TO_KILL

# Immediately watch
kubectl get pods -l app=hello --watch
```

Within seconds, you'll see a new Pod appear. The ReplicaSet noticed the count dropped from 3 to 2 and immediately spun up a replacement to get back to 3. **You did not have to do anything.**

Press `Ctrl+C` when you've seen it.

### Step 5 — Scale horizontally

The audience grew. Add more musicians:

```bash
kubectl scale deployment hello-deploy --replicas=5
kubectl get pods -l app=hello
```

Or, the better long-term habit — edit the YAML file (change `replicas: 3` to `replicas: 5`) and re-apply:

```bash
kubectl apply -f hello-deployment.yaml
```

> 🧠 **Why edit the file is better:** The YAML in Git is your source of truth. If you `kubectl scale` and someone later re-applies the old YAML, your scaling change disappears silently. Always change the file.

Scale back down:

```bash
kubectl scale deployment hello-deploy --replicas=3
```

### Step 6 — Expose the app internally with a ClusterIP Service

Right now your Pods have IPs, but those IPs change every time a Pod is replaced. We need a stable address.

Create `hello-service-clusterip.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
spec:
  type: ClusterIP
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

**The key line is `selector: app: hello`.** This Service will load-balance traffic to *any Pod with that label* — which happens to be the 3 Pods our Deployment created.

Apply it:

```bash
kubectl apply -f hello-service-clusterip.yaml
kubectl get service hello-svc
```

You'll see a `CLUSTER-IP` like `10.0.123.45`. **That IP will stay the same for the life of the Service**, even as the underlying Pods get replaced.

**Test it from inside the cluster:**

```bash
kubectl run tmp-curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s http://hello-svc/
```

You should see the nginx welcome HTML. The `--rm` flag deletes the temporary Pod when you exit.

> 🧠 **Notice:** You used the *name* `hello-svc`, not the IP. Kubernetes has built-in DNS — every Service gets a DNS name like `<service-name>.<namespace>.svc.cluster.local`. Inside the same namespace, just `hello-svc` works.

### Step 7 — Expose the app to the internet with a LoadBalancer Service

Time to let the public hear the orchestra. Create `hello-service-lb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-public
spec:
  type: LoadBalancer
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

Apply it:

```bash
kubectl apply -f hello-service-lb.yaml
```

**Watch for the public IP to be assigned:**

```bash
kubectl get service hello-public --watch
```

The `EXTERNAL-IP` column will show `<pending>` for 30–90 seconds while AKS provisions a real Azure Load Balancer behind the scenes. When it shows an actual IP, press `Ctrl+C`.

**Hit it from your laptop:**

```bash
EXTERNAL_IP=$(kubectl get svc hello-public -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXTERNAL_IP/
```

Or just paste the IP into your browser. **Your application is now on the public internet.** 🌍

> 💰 **Cost note:** A `LoadBalancer` Service provisions a real Azure Load Balancer. It costs a few cents per hour. We'll clean it up at the end. In Lab 4, you'll learn why production environments use Ingress instead of one LoadBalancer per app.

### Step 8 — Perform a rolling update

The artistic director announces: *"We're upgrading from nginx 1.27 to 1.28. Do it without silence."*

Edit `hello-deployment.yaml` and change the image tag:

```yaml
image: nginx:1.28-alpine
```

Apply the change:

```bash
kubectl apply -f hello-deployment.yaml
```

**Watch the rolling update happen:**

```bash
kubectl rollout status deployment/hello-deploy
kubectl get pods -l app=hello --watch
```

You'll see Kubernetes create new Pods one or two at a time, wait for them to be healthy, then terminate old ones. **At no point does the count drop below the level needed to serve traffic.** The Service keeps routing to whatever Pods are healthy. Audience hears no silence.

Inspect the rollout history:

```bash
kubectl rollout history deployment/hello-deploy
```

### Step 9 — Roll back when something breaks

Let's simulate a bad release. Edit `hello-deployment.yaml` and use a deliberately broken image tag:

```yaml
image: nginx:this-tag-does-not-exist
```

Apply:

```bash
kubectl apply -f hello-deployment.yaml
kubectl get pods -l app=hello
```

You'll see new Pods stuck in `ImagePullBackOff` — the new musicians can't find their sheet music. But notice: **the old Pods are still running and serving traffic**. The Deployment is conservative; it won't terminate healthy old Pods until new ones are healthy.

**Roll back:**

```bash
kubectl rollout undo deployment/hello-deploy
kubectl rollout status deployment/hello-deploy
kubectl get pods -l app=hello
```

You're back to the previous working version, with zero downtime. **This is the single most underrated feature of Kubernetes.**

> ⚠️ **Don't forget:** Update your YAML file back to the working tag (`nginx:1.28-alpine`) so it matches reality. The cluster and Git should always agree.

---

## ✅ Validation Checklist

```bash
kubectl get deployment hello-deploy        # READY 3/3
kubectl get rs                              # ReplicaSet with 3/3 ready
kubectl get pods -l app=hello               # 3 Pods Running
kubectl get svc                             # Both services present, LB has external IP
kubectl rollout history deployment/hello-deploy  # At least 2 revisions
curl http://<EXTERNAL_IP>/                  # Returns nginx welcome page
```

---

## 🧹 Cleanup

```bash
# Delete just this lab's resources (recommended if continuing to Lab 3)
kubectl delete namespace lab02

# OR delete the entire cluster
az group delete --name rg-aks-lab01 --yes --no-wait
```

> ⚠️ **Important:** Deleting the namespace also deletes the LoadBalancer Service, which releases the Azure Load Balancer and stops billing for it. Don't leave LoadBalancer Services running overnight.

---

## 🎓 What You Actually Learned

- **You almost never create Pods directly.** You create Deployments, which create ReplicaSets, which create Pods. This three-layer chain gives you self-healing, scaling, and safe upgrades for free.
- **Labels are how Kubernetes connects loosely coupled things.** A Service doesn't know or care which specific Pods exist — it just sends traffic to anyone wearing the right badge. This decoupling is what makes Kubernetes flexible.
- **Services give you stability over a moving target.** Pod IPs change constantly; Service IPs and DNS names don't.
- **`LoadBalancer` is real Azure infrastructure.** Each one provisions a frontend on an Azure Load Balancer. In production, you don't want one per app — that's why Ingress (Lab 4) exists.
- **Rolling updates and rollbacks are first-class features.** Most outages in non-Kubernetes systems come from deploys gone wrong; here, `kubectl rollout undo` gets you home in seconds.
- **Always change the YAML, not the cluster.** Imperative `kubectl scale` and `kubectl edit` are fine for emergencies, but the YAML file in Git is the source of truth.

---

## 🤔 Reflection Questions

1. If you delete a ReplicaSet directly (not the Deployment), what happens? What about the other way around?
2. Why does a Deployment create a *new* ReplicaSet during an update instead of modifying the existing one?
3. What would happen if two Deployments used the same `selector.matchLabels`?
4. The Service uses a label selector. What happens to traffic if you accidentally remove the label from a running Pod?
5. Why is `targetPort` separate from `port` in the Service spec?

---

## 📚 Further Reading

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service Types Explained](https://kubernetes.io/docs/concepts/services-networking/service/)
- [AKS Load Balancer Standard](https://learn.microsoft.com/en-us/azure/aks/load-balancer-standard)

---

## ➡️ What's Next

In **Lab 3**, we'll address a problem you may already be thinking about: *"Where do I put configuration and secrets?"* You'll learn about **ConfigMaps** (the program notes for the orchestra — non-secret config), **Secrets** (the conductor's locked safe — passwords and tokens), and **Persistent Volumes** (the music library that survives even when musicians come and go). By the end, your apps will be able to read configuration, hold credentials, and store data that outlives any individual Pod.

---

*Author: Prince Chime · Series: Kubernetes on AKS*
