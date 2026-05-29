# Lab 10: GitOps with ArgoCD + Multi-Tenant Platform

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🔴 Enterprise
> **Estimated Time:** 150–180 minutes
> **Prerequisite:** Labs 1–9 completed, plus a Git provider account (GitHub, GitLab, or Azure DevOps)

---

## 🎼 ELI5 — What Are We Actually Doing?

Until now, you've been the conductor's personal assistant — typing commands, applying YAMLs, running `helm upgrade`. It works, but it doesn't scale: you've been touching the cluster directly, there's no audit trail of *who changed what when*, and on Friday at 5 PM when you push a fix, no one else knows it happened.

Real enterprises don't operate this way. They follow a model called **GitOps**:

> **The Git repository is the single source of truth. The cluster is just a reflection of Git. Humans never touch the cluster directly.**

Think of it as the orchestra's master library replacing the conductor's improv. Every change to the performance — new musicians, a different tempo, a venue swap — is first written into the official score in the library. A robot called **ArgoCD** watches the library 24/7 and updates the live performance to match whatever the library currently says. Any drift between library and stage is automatically corrected. Every change has an author, a timestamp, and a reason (the Git commit). Rolling back is just `git revert`.

The benefits are massive:
- **Audit trail for free** — every cluster state is in Git history
- **Pull request reviews** — bad changes are caught before they hit prod
- **Easy disaster recovery** — lose the cluster? Re-create it, point ArgoCD at the repo, the cluster reconstructs itself
- **Multi-cluster fleet management** — one Git repo can drive 50 clusters
- **Real least-privilege** — humans don't need cluster admin; only ArgoCD does

In this final lab, you'll build a **mini multi-tenant platform**: ArgoCD running in your cluster, two simulated tenant teams (`team-alpha`, `team-beta`) each with their own namespace and Git directory, and a single ArgoCD `ApplicationSet` that automatically deploys whatever those teams put into Git. This is — at scale — exactly how internal platforms run at every cloud-native company.

---

## 🎯 Learning Objectives

- Install ArgoCD via Helm
- Connect ArgoCD to a Git repository
- Deploy applications declaratively as `Application` resources
- Use the App-of-Apps pattern to bootstrap multiple apps from one root
- Use `ApplicationSet` to template and auto-discover apps across many directories
- Configure sync policies (automated, self-healing, prune)
- Implement multi-tenant isolation with AppProjects and RBAC
- Understand sync waves and resource hooks for safe rollouts
- Connect ArgoCD's UI and CLI to your cluster
- Bootstrap a fresh cluster from Git (the disaster recovery scenario)

---

## 📋 Prerequisites

| Item | Required | Check |
|------|----------|-------|
| AKS cluster | Yes | `kubectl get nodes` |
| Helm 3 | Yes | `helm version` |
| A Git account (GitHub, GitLab, Azure DevOps) | Yes | You'll create one repo |
| `git` CLI installed and configured | Yes | `git --version` |
| (Optional) Labs 4 (ingress) and 9 (observability) installed | Recommended | For full demo |

---

## 🧠 Key Concepts

**GitOps** — A delivery pattern where Git is the source of truth and a controller in the cluster reconciles to match Git.

**ArgoCD** — A Kubernetes-native GitOps controller. Watches Git, syncs to cluster, reports drift.

**Application** — ArgoCD's main resource. Says *"deploy this Git path to this cluster/namespace using this method (Helm/Kustomize/raw YAML)."*

**ApplicationSet** — A generator that creates Applications dynamically. *"For every directory under `/teams/`, create an Application."* Eliminates copy-paste at scale.

**AppProject** — A multi-tenant boundary. Defines what a team is allowed to deploy, where, and from which Git repos. Critical for safety.

**Sync** — Apply Git's desired state to the cluster.

**Sync policy** — When/how to sync: manual, automated, with self-heal, with prune.

**Self-heal** — If someone manually edits the cluster (via `kubectl edit`), ArgoCD reverts it back to match Git.

**Prune** — When a resource is deleted from Git, delete it from the cluster too.

**Sync wave** — An annotation that controls ordering. Lower waves apply first. Useful for ensuring CRDs exist before custom resources.

**App-of-Apps pattern** — One ArgoCD Application that contains other ArgoCD Applications as its resources, recursively bootstrapping a whole platform.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Create your GitOps repository

Create a new public repository on GitHub (or your preferred provider) called `aks-gitops-lab`. Clone it locally:

```bash
cd ~
git clone https://github.com/<YOUR_USERNAME>/aks-gitops-lab.git
cd aks-gitops-lab
```

Set up the directory structure that the platform will follow:

```bash
mkdir -p {bootstrap,platform/argocd-projects,teams/team-alpha,teams/team-beta}

cat > README.md <<'EOF'
# AKS GitOps Lab

Single source of truth for the AKS cluster.

```
.
├── bootstrap/                       # The root ArgoCD App
├── platform/                        # Platform-team-managed config (AppProjects, RBAC, etc.)
└── teams/                           # One directory per tenant
    ├── team-alpha/                  # Anything team-alpha puts here gets deployed
    └── team-beta/
```
EOF

git add .
git commit -m "Initial directory structure"
git push
```

### Step 2 — Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd
```

Create `argocd-values.yaml`:

```yaml
configs:
  params:
    server.insecure: true   # we'll port-forward; in prod, terminate TLS at Ingress
  cm:
    timeout.reconciliation: 60s
server:
  service:
    type: ClusterIP
controller:
  resources:
    requests: { cpu: 200m, memory: 512Mi }
    limits:   { cpu: 500m, memory: 1Gi }
```

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --values argocd-values.yaml \
  --version 7.6.0

kubectl get pods -n argocd -w
# Wait for all 7 pods to be Running, then Ctrl+C
```

### Step 3 — Access the ArgoCD UI

Get the initial admin password:

```bash
ARGO_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d)
echo "Password: $ARGO_PASSWORD"
```

Port-forward and log in:

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:80
```

Open https://localhost:8080 (accept the self-signed cert), username `admin`, password from above.

You'll see an empty ArgoCD dashboard. Time to give it something to do.

### Step 4 — Install the ArgoCD CLI (recommended)

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Login
argocd login localhost:8080 \
  --insecure \
  --username admin \
  --password "$ARGO_PASSWORD"
```

### Step 5 — Create AppProjects for tenant isolation

Each AppProject defines a tenant boundary. Create `platform/argocd-projects/team-alpha-project.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha
  namespace: argocd
spec:
  description: Team Alpha — frontend apps
  sourceRepos:
    - 'https://github.com/<YOUR_USERNAME>/aks-gitops-lab.git'
  destinations:
    - namespace: 'team-alpha-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  roles:
    - name: developer
      policies:
        - p, proj:team-alpha:developer, applications, get, team-alpha/*, allow
        - p, proj:team-alpha:developer, applications, sync, team-alpha/*, allow
```

Create `platform/argocd-projects/team-beta-project.yaml` (same structure, swap "alpha" for "beta"):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-beta
  namespace: argocd
spec:
  description: Team Beta — backend apps
  sourceRepos:
    - 'https://github.com/<YOUR_USERNAME>/aks-gitops-lab.git'
  destinations:
    - namespace: 'team-beta-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

> 🧠 **The destination restriction is critical.** Team Alpha can only deploy to `team-alpha-*` namespaces. They cannot accidentally (or deliberately) deploy to `kube-system`, `argocd`, or another team's namespace. This is enforcement at the platform level, not on trust.

### Step 6 — Add example apps for each team

Create `teams/team-alpha/web-app.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha-prod
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpha-web
  namespace: team-alpha-prod
spec:
  replicas: 2
  selector: { matchLabels: { app: alpha-web } }
  template:
    metadata:
      labels: { app: alpha-web }
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:1.0.0
          args: ["-text=Hello from TEAM ALPHA via GitOps", "-listen=:5678"]
          ports:
            - containerPort: 5678
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits:   { cpu: 200m, memory: 128Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: alpha-web
  namespace: team-alpha-prod
spec:
  selector: { app: alpha-web }
  ports:
    - port: 80
      targetPort: 5678
```

Create `teams/team-beta/api.yaml` similarly:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-beta-prod
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-api
  namespace: team-beta-prod
spec:
  replicas: 3
  selector: { matchLabels: { app: beta-api } }
  template:
    metadata:
      labels: { app: beta-api }
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:1.0.0
          args: ["-text=Hello from TEAM BETA API via GitOps", "-listen=:5678"]
          ports:
            - containerPort: 5678
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits:   { cpu: 200m, memory: 128Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: beta-api
  namespace: team-beta-prod
spec:
  selector: { app: beta-api }
  ports:
    - port: 80
      targetPort: 5678
```

Commit and push:

```bash
git add .
git commit -m "Add team-alpha web app and team-beta api"
git push
```

### Step 7 — Apply the AppProjects

```bash
kubectl apply -f platform/argocd-projects/team-alpha-project.yaml
kubectl apply -f platform/argocd-projects/team-beta-project.yaml

# Or via the CLI
argocd proj list
```

> 🧠 **Notice we're applying these manually.** That's bootstrap chicken-and-egg — ArgoCD itself can manage these later (App of Apps), but you have to start somewhere.

### Step 8 — Create the ApplicationSet that auto-discovers tenant apps

Create `bootstrap/tenant-appset.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenants
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/<YOUR_USERNAME>/aks-gitops-lab.git
        revision: HEAD
        directories:
          - path: teams/*
  template:
    metadata:
      name: '{{path.basename}}'         # team-alpha, team-beta
    spec:
      project: '{{path.basename}}'      # references the AppProject by team name
      source:
        repoURL: https://github.com/<YOUR_USERNAME>/aks-gitops-lab.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}-prod'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

```bash
kubectl apply -f bootstrap/tenant-appset.yaml
```

**What just happened:**

- The ApplicationSet's `git generator` scanned `teams/*` in your repo and found `team-alpha` and `team-beta`.
- For each one, it generated an `Application` named after the team (using `{{path.basename}}`).
- Each Application points at its team's directory in Git.
- The sync policy says: auto-sync, self-heal cluster drift, and prune resources removed from Git.

In the ArgoCD UI, you'll see two Applications appear, sync, and turn green.

```bash
argocd app list
kubectl get pods -n team-alpha-prod
kubectl get pods -n team-beta-prod
```

**Both apps deployed automatically. From a Git push.**

### Step 9 — The GitOps loop in action

**Make a change via Git, watch the cluster update:**

In your repo, edit `teams/team-alpha/web-app.yaml` and change `replicas: 2` to `replicas: 5`. Commit and push:

```bash
git add teams/team-alpha/web-app.yaml
git commit -m "team-alpha: scale web to 5 replicas"
git push
```

In the ArgoCD UI (or via CLI), within ~3 minutes (default poll interval) you'll see the Application detect the change and sync. Or force a sync:

```bash
argocd app sync team-alpha
kubectl get pods -n team-alpha-prod
```

5 Pods now running. **You changed production by writing a Git commit.** No `kubectl`, no `helm upgrade`. The change is reviewable, attributable, and reversible.

### Step 10 — Test self-heal

This is the magical bit. Manually break the cluster:

```bash
kubectl scale deployment alpha-web --replicas=99 -n team-alpha-prod
kubectl get deploy -n team-alpha-prod
# replicas: 99
```

Wait ~60 seconds. Check again:

```bash
kubectl get deploy -n team-alpha-prod
# replicas: 5 (back to whatever Git says)
```

ArgoCD detected drift between cluster and Git, and **silently reverted your manual change**. In production, this prevents the *"someone made an emergency manual change and forgot to update Git"* problem that has burned every operations team in history.

### Step 11 — Test prune

Delete `teams/team-beta/api.yaml` from Git:

```bash
git rm teams/team-beta/api.yaml
git commit -m "team-beta: deprecate api"
git push
```

Wait or trigger a sync:

```bash
argocd app sync team-beta
```

The Deployment, Service, and even the namespace are deleted from the cluster. **What's not in Git is not in the cluster.** Restore by reverting the commit:

```bash
git revert HEAD
git push
argocd app sync team-beta
```

Resources reappear. **Rollback = git revert.** This is the cleanest deployment system you'll ever use.

### Step 12 — App-of-Apps: bootstrapping the platform itself

Let's go meta. Create `bootstrap/root.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_USERNAME>/aks-gitops-lab.git
    targetRevision: HEAD
    path: bootstrap            # root manages everything in bootstrap/
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Move/keep the ApplicationSet under `bootstrap/`:

```bash
# Already there: bootstrap/tenant-appset.yaml

# Also add the AppProjects to bootstrap so root manages them:
mv platform/argocd-projects/*.yaml bootstrap/
git add .
git commit -m "Move AppProjects under bootstrap for App-of-Apps"
git push
```

Apply the root:

```bash
kubectl apply -f bootstrap/root.yaml
```

**You now have a single Application that manages the AppProjects, which set the boundaries, and the ApplicationSet, which discovers tenant apps. The whole platform is described by one root Application.**

### Step 13 — The disaster recovery test

The real reward of GitOps:

```bash
# Pretend everything caught fire — delete the cluster
az group delete --name rg-aks-lab07 --yes

# Recreate
az group create --name rg-aks-lab10 --location westeurope
az aks create --resource-group rg-aks-lab10 --name aks-lab10 \
  --node-count 3 --node-vm-size Standard_B4ms \
  --enable-managed-identity --generate-ssh-keys

az aks get-credentials -g rg-aks-lab10 -n aks-lab10

# Reinstall ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --values argocd-values.yaml --version 7.6.0

# Apply ONLY the root Application
kubectl apply -f bootstrap/root.yaml
```

**Wait 5 minutes. The entire platform reconstitutes itself.** AppProjects, ApplicationSets, tenant apps, namespaces — everything that was in Git comes back. This is what enterprises mean when they say "infrastructure as code with disaster recovery."

### Step 14 — Sync waves for ordered rollouts

Sometimes resources must be applied in order: CRDs before custom resources, namespaces before Pods, etc. ArgoCD respects an annotation:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # apply earlier (lower = earlier)
```

Common patterns:
- `-1` for namespaces and CRDs
- `0` (default) for typical resources
- `1` for things that need their dependencies in place
- Higher waves for post-deploy jobs (migrations, smoke tests)

### Step 15 — Hook a sync to a job (database migration pattern)

A classic enterprise need: run a migration *before* deploying the new app. Use `PreSync` hooks:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: your-app:latest
          command: ["./migrate.sh"]
```

Each sync runs the migration first, fails fast if it fails, and only then deploys the new app. Same pattern works for `Sync`, `PostSync`, `SyncFail`.

---

## ✅ Validation Checklist

```bash
argocd app list                                # 2+ Applications, all Synced and Healthy
kubectl get pods -n team-alpha-prod            # alpha-web running
kubectl get pods -n team-beta-prod             # beta-api running
argocd app get root                            # root Application exists and Healthy

# Drift test
kubectl scale deployment alpha-web --replicas=99 -n team-alpha-prod
sleep 90
kubectl get deploy alpha-web -n team-alpha-prod  # Back to 5
```

---

## 🧹 Cleanup

```bash
# Remove the root (cascade-deletes everything it owns)
kubectl delete application root -n argocd
helm uninstall argocd -n argocd
kubectl delete namespace argocd team-alpha-prod team-beta-prod

# Or nuke the whole cluster
az group delete --name rg-aks-lab10 --yes --no-wait
```

> 💡 **Keep the Git repo.** It's a reusable starting template for future projects.

---

## 🎓 What You Actually Learned

- **Git is the source of truth.** Every cluster state, every team's deployment, every config — all of it lives in version control. The cluster is just a reflection.
- **ArgoCD is a closed-loop reconciler.** It compares "what Git says" with "what's running" and fixes the difference, continuously.
- **AppProjects are tenant boundaries.** Without them, anyone with ArgoCD access can deploy anything anywhere. With them, each team is sandboxed.
- **ApplicationSets eliminate boilerplate.** Instead of writing one `Application` per team and updating it forever, you describe the *pattern* and let ArgoCD discover what matches.
- **Self-heal eliminates drift.** Manual `kubectl edits` are silently reverted. This sounds annoying but is actually the feature — drift is how outages start.
- **Prune enforces declarativeness.** If it's not in Git, it shouldn't exist. No more orphan resources after a botched cleanup.
- **App-of-Apps bootstraps the platform itself.** One root Application can describe everything — AppProjects, network policies, observability, ingress controllers — recursively.
- **GitOps is a disaster recovery story.** Lose the cluster? `helm install argocd` + apply the root Application = the whole platform reconstitutes from Git in minutes.
- **Pull request reviews become deployment reviews.** The same code review process catches bad deploys before they happen. Less "oops I broke prod" energy across the org.
- **This is the modern enterprise model.** Every cloud-native company you've heard of operates this way at some level. Now you have hands-on experience with the real thing.

---

## 🤔 Reflection Questions

1. What's the failure mode if your Git provider goes down? Does ArgoCD keep working?
2. Self-heal would revert a hotfix made via `kubectl edit`. How do you handle a real emergency where you need to bypass Git?
3. The ApplicationSet uses `{{path.basename}}` — what happens if you create a folder `teams/.hidden/`?
4. Why are AppProjects more important in multi-tenant clusters than in single-tenant ones?
5. ArgoCD vs Flux — what's the difference at a high level? When might you choose Flux instead?
6. Where do you store secrets in a GitOps repo? (Hint: not in plain text — research SealedSecrets, External Secrets Operator, or Sops.)
7. How do you do canary releases with ArgoCD? (Hint: look up Argo Rollouts.)

---

## 📚 Further Reading

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ApplicationSet Generators](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/)
- [GitOps Principles (OpenGitOps)](https://opengitops.dev/)
- [Argo Rollouts (canary, blue/green)](https://argo-rollouts.readthedocs.io/)
- [External Secrets Operator (Azure Key Vault integration)](https://external-secrets.io/latest/provider/azure-key-vault/)
- [Comparing ArgoCD vs Flux](https://www.cncf.io/blog/2023/07/05/argo-vs-flux-comparing-the-leading-gitops-controllers/)

---

## 🎓 Series Wrap-Up — What You've Built

Across 10 labs, you've gone from "I have an Azure subscription" to running a multi-tenant, observable, secure, scalable, GitOps-driven Kubernetes platform on AKS. Specifically:

| Lab | What you can now do |
|-----|----------------------|
| 1 | Provision and connect to AKS, deploy Pods |
| 2 | Build self-healing, scalable, externally-reachable apps |
| 3 | Manage configuration, secrets, and persistent state |
| 4 | Expose apps publicly with one front door and free TLS |
| 5 | Package apps as reusable Helm charts |
| 6 | Manage multi-environment deployments cleanly with Kustomize |
| 7 | Lock the cluster down with RBAC, network policies, and Pod security |
| 8 | Scale dynamically on metrics and events, including to zero |
| 9 | See what's happening with metrics, logs, and alerts |
| 10 | Run everything via Git — reviewable, auditable, recoverable |

**This is genuinely production-grade knowledge.** It's the same toolchain (with bigger numbers and stricter controls) that runs at most cloud-native enterprises today.

### Where to Go Next

- **Service mesh (Istio, Linkerd, Cilium Service Mesh)** — east-west traffic control, mTLS, advanced traffic shaping
- **Distributed tracing (Tempo, Jaeger, OpenTelemetry)** — the third pillar of observability
- **Argo Rollouts** — canary and blue/green deployments
- **Crossplane** — manage Azure infrastructure (databases, storage accounts) from inside Kubernetes
- **CKA / CKAD certifications** — formal validation of what you now know
- **Backstage** — internal developer portal (the next frontier in IDPs, which connects directly to your day-to-day work)

You started this series asking how to learn AKS. You're finishing it ready to build platforms.

🎼 *And the orchestra plays on.*

---

*Author: Prince Chime · Series: Kubernetes on AKS — Complete*
*If this series helped you, subscribe to the channel: [@princeugochime](https://youtube.com/@princeugochime)*
