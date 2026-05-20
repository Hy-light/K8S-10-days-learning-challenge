# Lab 3: ConfigMaps, Secrets & Persistent Storage

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟢 Beginner → 🟡 Intermediate
> **Estimated Time:** 75–90 minutes
> **Prerequisite:** Labs 1–2 completed

---

## 🎼 ELI5 — What Are We Actually Doing?

Your orchestra is now reliable and reachable (Lab 2). But three problems remain:

1. **Where do the musicians get their program notes?** Not the sheet music itself — the surrounding details: tempo markings, the venue's acoustic settings, which language to print the program in. These change between performances but aren't part of the music. In Kubernetes, this is a **ConfigMap**.

2. **Where does the conductor keep the locked safe?** The combination to the building, the bank account for paying musicians, the encrypted backstage pass codes. These must never be written on a poster or printed in the program. In Kubernetes, this is a **Secret**.

3. **Where does the orchestra's music library live?** Decades of sheet music, recordings, and performance notes. Individual musicians come and go, but the library must persist. If the entire orchestra is replaced, the new one should still find every score. In Kubernetes, this is a **Persistent Volume**.

In this lab, you'll inject configuration into your Pods three different ways, store secrets safely (and learn why the default isn't *that* safe), and attach durable storage that survives Pod restarts and rescheduling.

---

## 🎯 Learning Objectives

By the end of this lab, you will:

- Create ConfigMaps from literal values, files, and YAML
- Inject ConfigMap data into Pods as environment variables AND as mounted files
- Create and use Secrets (and understand they're only base64-encoded by default)
- Understand the StorageClass / PersistentVolume / PersistentVolumeClaim model
- Attach Azure Disk storage to a Pod that survives restarts
- Know when to use Azure Disk vs Azure Files

---

## 📋 Prerequisites

| Item | Required | Notes |
|------|----------|-------|
| AKS cluster | Yes | From Lab 1 |
| Lab 2 namespace cleaned up | Recommended | Or use a fresh `lab03` namespace |

---

## 🧠 Key Concepts

**ConfigMap** — Key-value data for non-sensitive configuration. Plain text. Visible to anyone with read access to the namespace.

**Secret** — Key-value data for sensitive information. **By default, only base64-encoded, NOT encrypted at rest** (unless you enable encryption providers — more in Lab 7). Treat them like config that's *slightly* better hidden, not like a vault.

**StorageClass** — A "menu item" describing a type of storage you can request (e.g., "fast SSD," "cheap HDD," "shared file storage"). AKS comes with several built-in.

**PersistentVolume (PV)** — An actual chunk of storage in the cluster (e.g., a 10 GB Azure Disk). Usually provisioned automatically when requested.

**PersistentVolumeClaim (PVC)** — A request for storage. *"I need 5 GB of fast SSD."* Kubernetes finds or creates a matching PV and binds them.

**Mental model:** StorageClass is the menu, PVC is the order, PV is the dish that arrives.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Set up a new namespace

```bash
kubectl create namespace lab03
kubectl config set-context --current --namespace=lab03
```

### Step 2 — Create a ConfigMap (three ways)

**Way A — From literal values (quick, imperative):**

```bash
kubectl create configmap app-config-literal \
  --from-literal=APP_ENV=production \
  --from-literal=APP_REGION=westeurope \
  --from-literal=LOG_LEVEL=info
```

**Way B — From a file:**

```bash
cat > app.properties <<EOF
welcome.message=Hello from Prince's lab
feature.dark_mode=true
feature.beta=false
EOF

kubectl create configmap app-config-file --from-file=app.properties
```

**Way C — From a YAML manifest (the production way):**

Create `configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_REGION: "westeurope"
  LOG_LEVEL: "info"
  welcome.html: |
    <html>
      <body>
        <h1>Hello from Prince's Lab</h1>
        <p>Region: westeurope · Env: production</p>
      </body>
    </html>
```

```bash
kubectl apply -f configmap.yaml
kubectl get configmap
kubectl describe configmap app-config
```

> 🧠 **Notice the `welcome.html` key.** ConfigMap values can be entire files. We'll mount this as an actual file inside the Pod.

### Step 3 — Inject ConfigMap data into a Pod

There are two main ways: as environment variables, and as mounted files. We'll do both in one Deployment.

Create `app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          # Method 1: env vars from ConfigMap keys
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: LOG_LEVEL
          # Method 2: mount welcome.html as a file in the nginx web root
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
      volumes:
        - name: web-content
          configMap:
            name: app-config
            items:
              - key: welcome.html
                path: index.html
```

Apply and verify:

```bash
kubectl apply -f app-deployment.yaml
kubectl get pods -l app=web

# Check env vars are injected
POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep -E "APP_ENV|LOG_LEVEL"

# Check the file is mounted
kubectl exec $POD -- cat /usr/share/nginx/html/index.html

# Reach the app
kubectl port-forward $POD 8080:80
# In another terminal: curl http://localhost:8080
```

You should see your custom HTML. **Configuration changed without rebuilding the container image.**

### Step 4 — Update a ConfigMap and watch what happens

Edit `configmap.yaml` and change the welcome message:

```yaml
welcome.html: |
  <html><body><h1>UPDATED MESSAGE</h1></body></html>
```

```bash
kubectl apply -f configmap.yaml

# Wait ~60 seconds, then check the file inside the running Pod
kubectl exec $POD -- cat /usr/share/nginx/html/index.html
```

**The mounted file updates automatically** (within a minute or so — kubelet syncs ConfigMap-mounted files periodically).

⚠️ **But environment variables do NOT update.** They're frozen at Pod start time. To pick up changes to env vars, you must restart the Pod:

```bash
kubectl rollout restart deployment/web
```

> 🧠 **Real-world implication:** Mount config as files when you want hot reloads (and your app supports re-reading them). Use env vars for values that won't change without a restart anyway.

### Step 5 — Create a Secret

Secrets work almost identically to ConfigMaps, but Kubernetes treats them with a bit more care (not stored in `etcd` plain text *if* encryption-at-rest is enabled, redacted from `kubectl describe` output, etc.).

**Imperative:**

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASSWORD='S3cur3-P@ssw0rd!'
```

**Or via YAML** — but you must base64-encode values yourself. Create `db-secret.yaml`:

```bash
echo -n 'appuser' | base64          # YXBwdXNlcg==
echo -n 'S3cur3-P@ssw0rd!' | base64  # UzNjdXIzLVBAc3N3MHJkIQ==
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-yaml
type: Opaque
data:
  DB_USER: YXBwdXNlcg==
  DB_PASSWORD: UzNjdXIzLVBAc3N3MHJkIQ==
```

> ⚠️ **Critical truth bomb:** Base64 is not encryption. Anyone with `kubectl get secret -o yaml` access can decode it instantly. In production, you protect Secrets with:
> 1. **RBAC** (Lab 7) — restrict who can read them.
> 2. **Encryption at rest** — enable etcd encryption providers (Azure Key Vault encryption for AKS).
> 3. **External secret managers** — Azure Key Vault + the Secrets Store CSI driver (we'll touch on this in Lab 7).
>
> The default "Secret" in Kubernetes is *better than nothing*, not "secure."

### Step 6 — Use the Secret in a Pod

Add this `env` block to the container in `app-deployment.yaml` (alongside the existing `APP_ENV` etc.):

```yaml
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: DB_USER
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: DB_PASSWORD
```

Apply and verify:

```bash
kubectl apply -f app-deployment.yaml
kubectl rollout status deployment/web

POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep DB_
```

You'll see the secret values — but only because you have exec access. A user with read-only access to the namespace cannot.

### Step 7 — Inspect AKS storage classes

```bash
kubectl get storageclass
```

You'll see several built-in classes on AKS, typically including:

- `default` / `managed-csi` — Azure Disk (block storage, attached to one Pod at a time)
- `managed-csi-premium` — Azure Premium SSD
- `azurefile-csi` — Azure Files (shared file storage, multiple Pods can mount it)

**Mental rule of thumb:**
- **Azure Disk** = fast, single-writer (databases, single-Pod state)
- **Azure Files** = slower, multi-writer (shared assets, multi-Pod read-write)

### Step 8 — Request persistent storage with a PVC

Create `pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: managed-csi
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
kubectl get pv
```

You'll see the PVC in `Pending` state at first, then `Bound` once a matching PV is dynamically provisioned by AKS (usually within seconds). **A real Azure Managed Disk now exists in your subscription.**

### Step 9 — Attach the PVC to a Pod and prove persistence

Create `stateful-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-writer
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Pod started at $(date)" >> /data/log.txt
          echo "Pod hostname: $(hostname)" >> /data/log.txt
          tail -f /dev/null
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

```bash
kubectl apply -f stateful-pod.yaml
kubectl wait --for=condition=Ready pod/data-writer --timeout=120s

# See what was written
kubectl exec data-writer -- cat /data/log.txt
```

**Now the proof — kill the Pod and bring it back:**

```bash
kubectl delete pod data-writer
kubectl apply -f stateful-pod.yaml
kubectl wait --for=condition=Ready pod/data-writer --timeout=120s

kubectl exec data-writer -- cat /data/log.txt
```

You'll see **two entries** — the original write and the new one. The data survived because it's on the Azure Disk, not in the Pod. The Pod is ephemeral; the volume isn't.

> 🧠 **Why this matters:** This is exactly how databases work in Kubernetes. The container can crash, get rescheduled to another node, get upgraded — and the data stays.

---

## ✅ Validation Checklist

```bash
kubectl get configmap                       # 3 ConfigMaps
kubectl get secret                          # db-credentials present
kubectl get pvc                             # data-pvc Bound
kubectl get pv                              # Auto-created PV exists
kubectl exec data-writer -- cat /data/log.txt  # Multiple entries after restart
```

---

## 🧹 Cleanup

```bash
# Delete the namespace (releases everything inside it)
kubectl delete namespace lab03

# IMPORTANT: deleting the namespace deletes the PVC, which by default
# deletes the underlying Azure Disk. Verify in Azure portal that no orphan
# disks remain in the AKS-managed resource group.
```

> ⚠️ **Reclaim policy gotcha:** The default reclaim policy on dynamically provisioned PVs is `Delete`, meaning the Azure Disk is destroyed when the PVC goes. For production data, set the StorageClass `reclaimPolicy: Retain` so accidental PVC deletion doesn't lose data.

---

## 🎓 What You Actually Learned

- **Configuration belongs outside the image.** Bake config into images and you have to rebuild for every environment change. ConfigMaps and Secrets keep config separate, so the same image runs in dev, staging, and prod.
- **Two ways to consume config:** environment variables (frozen at Pod start) and mounted files (auto-update). Pick based on your app's behavior.
- **Secrets are not secure by default.** Base64 is encoding, not encryption. Real protection comes from RBAC + encryption at rest + external secret stores.
- **The PVC abstraction decouples your app from storage details.** The Pod says "I need 5 GB"; AKS handles the Azure Disk lifecycle.
- **Persistent storage outlives Pods.** This is the foundation for stateful workloads — databases, message queues, caches.
- **Reclaim policy matters.** `Delete` (default) means losing the PVC loses the data. `Retain` preserves the disk for manual recovery.

---

## 🤔 Reflection Questions

1. If your app reads config from an env var, and you update the ConfigMap, why doesn't the change take effect?
2. What's the difference between `accessModes: ReadWriteOnce` and `ReadWriteMany`? When would you need each?
3. If two Pods on different nodes both try to mount the same `ReadWriteOnce` Azure Disk PVC, what happens?
4. Why is base64 used for Secret values if it's not encryption?
5. In what scenarios is Azure Files the right choice over Azure Disk?

---

## 📚 Further Reading

- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secrets — Good Practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [AKS Storage Concepts](https://learn.microsoft.com/en-us/azure/aks/concepts-storage)
- [Azure Key Vault Provider for Secrets Store CSI Driver](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)

---

## ➡️ What's Next

In **Lab 4**, we'll fix a problem you may have noticed: every public-facing Service requires its own Azure Load Balancer, which is wasteful and expensive. You'll deploy an **Ingress Controller** (NGINX) — one front door for all your apps with smart routing — and add **automatic TLS certificates** with cert-manager and Let's Encrypt. Your apps will be reachable at `https://yourapp.example.com` with proper certificates, all managed declaratively.

---

*Author: Prince Chime · Series: Kubernetes on AKS*
