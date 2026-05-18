# Lab 1: AKS Cluster Setup & Your First Pod Deployment

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟢 Beginner
> **Estimated Time:** 45–60 minutes
> **Cost Warning:** This lab provisions real Azure resources. Remember to clean up at the end (~$0.10/hour while running).

---

## 🎼 ELI5 — What Are We Actually Doing?

Imagine you're putting on a concert. You have musicians, instruments, sheet music, and a concert hall — but without someone to coordinate them, it's just noise. You need a **conductor**: someone who tells each musician when to play, fixes things when a violinist's string snaps mid-performance, and brings in a substitute if someone falls ill.

That conductor is **Kubernetes**.

In fact, the word _Kubernetes_ comes from Greek meaning "helmsman" or "pilot" — and the industry calls what it does **orchestration**. That's not marketing fluff; it's literally what's happening. Kubernetes coordinates many small running programs (musicians) so they perform together as a coherent application (a symphony).

In this lab, we're doing two things:

1. **Renting a concert hall from Microsoft** — Azure Kubernetes Service (AKS) is Microsoft's "managed" version of this orchestra setup. "Managed" means Microsoft hires the conductor for you (the control plane) and maintains the hall, while you focus on the music.
2. **Asking the conductor to cue a single musician** — a **Pod**. A Pod is the smallest unit in Kubernetes; it's basically _one musician playing_. We'll have one play a simple tune (an nginx web server that says "Hello").

Why start this small? Because before you conduct Beethoven's 9th, you need to know how to cue one violinist. Once you can reliably bring in one musician, everything else is just bigger combinations of the same idea.

---

## 🎯 Learning Objectives

By the end of this lab, you will:

- Understand what AKS is and how it differs from self-managed Kubernetes
- Provision an AKS cluster using the Azure CLI
- Connect `kubectl` to your cluster
- Deploy your first Pod using both imperative and declarative approaches
- Understand the difference between a Pod, a container, and a node
- Inspect cluster resources and read logs

---

## 📋 Prerequisites

| Tool                  | Version                 | Check Command              |
| --------------------- | ----------------------- | -------------------------- |
| Azure CLI             | 2.50+                   | `az --version`             |
| kubectl               | 1.28+                   | `kubectl version --client` |
| An Azure subscription | With Contributor rights | `az account show`          |

> 💡 **Tip:** If `kubectl` isn't installed, run `az aks install-cli` — the Azure CLI will install it for you.

---

## 🧠 Key Concepts (Read Before You Start)

**Cluster** — The whole concert hall, including the conductor and all sections. It has a brain (control plane) and performers (worker nodes).

**Node** — A single VM that hosts your workloads. Think of it as one section of the orchestra (e.g., the strings section) — a place where musicians sit and play.

**Pod** — The smallest unit you can deploy. Usually one container (one musician), sometimes a few that _must_ play together (like a violinist and their page-turner — they have to share a music stand).

**Container** — The actual running application (e.g., nginx, your Python app). Pods _contain_ containers, the way a musician _plays_ an instrument.

**kubectl** — The baton. It's how you (or the conductor) signal what should happen.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Log in and set your subscription

```bash
az login
az account set --subscription "<your-subscription-name-or-id>"
az account show --output table
```

### Step 2 — Create a resource group

A resource group is just a folder in Azure for keeping related things together — think of it as the binder where you keep all the paperwork for one concert.

```bash
az group create \
  --name rg-aks-lab01 \
  --location westeurope
```

> 🌍 Use a region close to you. For Lisbon, `westeurope` (Netherlands) or `francecentral` (Paris) are good options.

### Step 3 — Create the AKS cluster

This is the big one. We're booking a small, affordable concert hall suitable for a rehearsal.

```bash
az aks create \
  --resource-group rg-aks-lab01 \
  --name aks-lab01 \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --generate-ssh-keys \
  --tier free
```

**What each flag means in plain English:**

- `--node-count 2` → Give me 2 sections in the orchestra (2 worker VMs)
- `--node-vm-size Standard_B2s` → Use small, burstable VMs (~$30/month each, but we're deleting it today)
- `--enable-managed-identity` → Let the cluster authenticate to Azure services without us juggling passwords
- `--tier free` → Use the free control plane tier (no SLA, perfect for learning)

⏳ **This takes 5–10 minutes.** Go grab a coffee.

### Step 4 — Connect kubectl to your cluster

```bash
az aks get-credentials \
  --resource-group rg-aks-lab01 \
  --name aks-lab01
```

This downloads cluster credentials into `~/.kube/config` so `kubectl` (your baton) knows which orchestra to conduct.

**Verify the connection:**

```bash
kubectl get nodes
```

You should see 2 nodes in `Ready` state. If you do, **congratulations — your orchestra is seated and tuning up.** 🎉

### Step 5 — Deploy your first Pod (the imperative way)

The "imperative" way means _telling kubectl what to do_, like calling out cues verbally during rehearsal.

```bash
kubectl run hello-pod \
  --image=nginx:1.27-alpine \
  --port=80
```

**What happened?** You told Kubernetes: _"Cue a musician called `hello-pod` to play the nginx piece, and they'll be audible on port 80."_

**Watch them come in:**

```bash
kubectl get pods --watch
```

When `STATUS` becomes `Running`, press `Ctrl+C`.

### Step 6 — Inspect your Pod

```bash
# See basic info
kubectl get pod hello-pod

# See detailed info (events, IP, node it's on, etc.)
kubectl describe pod hello-pod

# See its logs (what the musician is "saying")
kubectl logs hello-pod

# Step onto the stage with the musician (like SSH-ing in)
kubectl exec -it hello-pod -- sh
# Once inside, run: curl localhost
# Type 'exit' to leave
```

### Step 7 — Deploy the same Pod the declarative way

The "declarative" way means _writing the sheet music in a YAML file_. **This is how real Kubernetes work happens** — imperative commands are just for quick experiments and warm-ups.

Create a file called `hello-pod-declarative.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod-declarative
  labels:
    app: hello
    owner: prince
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

**Apply it:**

```bash
kubectl apply -f hello-pod-declarative.yaml
kubectl get pods
```

🧠 **Why declarative is better:** The YAML file is your sheet music — your source of truth. You can commit it to Git, review it in a PR, and replay the same performance on any cluster. The imperative `kubectl run` from Step 5 is like humming a tune from memory — it leaves no record.

### Step 8 — Reach the Pod from your laptop

Pods aren't exposed externally by default — by default, the music stays inside the hall (we'll fix that properly in Lab 2 with Services). For now, use port-forwarding, which is like running a private audio cable from the stage to your seat:

```bash
kubectl port-forward pod/hello-pod-declarative 8080:80
```

Open another terminal:

```bash
curl http://localhost:8080
```

You should see the nginx welcome HTML. Press `Ctrl+C` in the port-forward terminal when done.

---

## ✅ Validation Checklist

Run these and confirm each one works:

```bash
kubectl get nodes                          # Both nodes Ready
kubectl get pods                           # Both pods Running
kubectl describe node | grep -A 5 Capacity # See node CPU/memory
kubectl top nodes                          # Live resource usage (may take a minute to populate)
```

---

## 🧹 Cleanup (IMPORTANT — Don't Skip)

```bash
# Send the musicians home (keep the hall for the next lab if you want)
kubectl delete pod hello-pod hello-pod-declarative

# OR tear down the whole concert hall to stop billing
az group delete --name rg-aks-lab01 --yes --no-wait
```

> ⚠️ **Reminder:** AKS clusters cost money even when idle — like paying rent on a hall whether or not the orchestra plays. If you're not moving to Lab 2 today, delete the resource group.

---

## Understanding Pod Resource Requests and Limits

> What does this part of my manifest mean?
>
> ```yaml
> resources:
>   requests:
>     cpu: "50m"
>     memory: "64Mi"
>   limits:
>     cpu: "200m"
>     memory: "128Mi"
> ```

**Answer:**

This `resources` section controls how much **CPU** and **memory** the container is allowed to use and how much will be reserved for it on the Kubernetes node.

### `requests` – The guaranteed minimum

- **`cpu: "50m"`** → 50 millicores = 0.05 of a CPU core.  
  The scheduler uses this to decide where to place the Pod. It reserves at least 50 millicores for this container, so it always gets that much compute time (even under load).
- **`memory: "64Mi"`** → 64 mebibytes (≈67 MB).  
  The node reserves this amount of RAM. The container will never be able to use less than this (if it tries, the OS may throttle it, but it won't be killed for going under).

### `limits` – The maximum allowed

- **`cpu: "200m"`** → 200 millicores = 0.2 of a CPU core.  
  The container cannot use more CPU than this. If it tries, Kubernetes **throttles** it (slows it down) – it won’t be killed.
- **`memory: "128Mi"`** → 128 mebibytes (≈134 MB).  
  This is a hard cap. If the container tries to allocate more than 128Mi of RAM, it will be **terminated (OOMKilled)** by the kernel. This prevents a single bad container from starving other Pods on the node.

### How they work together

- **Requests ≤ Limits** – Here, requests are lower than limits, which is common. The container is guaranteed 50m CPU / 64Mi RAM, but can burst up to 200m CPU / 128Mi RAM if the node has spare resources.
- If requests and limits are **equal**, the container gets **Guaranteed** QoS (Quality of Service), which is most stable.
- If requests < limits (like this example), the QoS class is **Burstable** – good for typical web apps that have normal and peak loads.

### Real‑world effect in your lab

- **Scheduling** – The node must have at least 50m CPU + 64Mi RAM _free_ (unreserved by other Pods) to place this Pod.
- **Runtime** – Your nginx will start quickly with its 64Mi guarantee. If traffic spikes, it can use extra CPU (up to 0.2 cores) and memory (up to 128Mi). Beyond 128Mi memory → container dies.
- **Node overcommit** – Because you set limits higher than requests, the node can run more Pods than its physical resources would allow (overcommit), betting they won’t all use their max at once.

### Units reminder

- **CPU** – `1` = 1 vCPU/core. Millicores: `1000m` = 1 core. You used `50m` and `200m`.
- **Memory** – `Ki` (kibibytes), `Mi` (mebibytes), `Gi` (gibibytes). `64Mi` = 64 × 1024² bytes.

**Important:** If you ever remove `limits` entirely, the container has no memory cap (could eat all node RAM) and can use unlimited CPU (throttling still applies but no hard limit). That’s dangerous in shared clusters – so always specify at least memory limits for production.

In your lab, this configuration is safe and typical for a lightweight nginx test Pod.

---

## 🎓 What You Actually Learned

- **AKS abstracts away the control plane** — you don't manage etcd, the API server, or the scheduler. Azure hires the conductor for you. You only pay for and manage worker nodes (the musicians' chairs).
- **Pods are ephemeral** — they're not designed to be long-lived. If a node dies, a bare Pod dies with it. (Lab 2 fixes this with Deployments — understudies who can step in.)
- **Imperative vs declarative** — you'll use imperative for debugging and exploration (calling cues), declarative (YAML) for everything that matters (sheet music).
- **kubectl is just an HTTP client** — every command you ran translated to an API call against the AKS control plane.
- **Resource requests and limits matter** — without them, a noisy Pod can starve its neighbors, like one musician playing so loudly the others can't be heard. We set them in Step 7.

---

## 🤔 Reflection Questions

Try answering these before moving to Lab 2:

1. What happens if you run `kubectl delete pod hello-pod` and then `kubectl get pods`? Why?
2. Why did we use `nginx:1.27-alpine` instead of just `nginx`?
3. If a node goes down, what happens to the Pods running on it?
4. What's the difference between `kubectl create` and `kubectl apply`?

---

## 📚 Further Reading

- [AKS Documentation — Core Concepts](https://learn.microsoft.com/en-us/azure/aks/concepts-clusters-workloads)
- [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

---

## ➡️ What's Next

In **Lab 2**, we'll fix the biggest weakness of bare Pods — they don't survive failure. You'll learn about **Deployments** (musicians with understudies who auto-replace them), **ReplicaSets** (the engine behind Deployments — like the personnel manager keeping the right number of musicians on stage), and **Services** (a stable address so the audience can always find the orchestra, even when individual musicians get swapped out).

---

_Author: Prince Chime · Series: Kubernetes on AKS_
