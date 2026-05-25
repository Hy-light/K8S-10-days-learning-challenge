# Lab 6: Kustomize for Multi-Environment Deployments

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟡 Intermediate
> **Estimated Time:** 75–90 minutes
> **Prerequisite:** Labs 1–5 completed

---

## 🎼 ELI5 — What Are We Actually Doing?

In Lab 5 (Helm), the touring orchestra brought a master score with blanks — and each venue filled in the blanks with their values sheet. That works, but it has a quirk: the master score is *written in a different language* (Go templates with `{{ }}` everywhere). When something looks wrong, you can't just read the score and understand what will be played — you have to first translate it.

Kustomize takes a completely different approach. The base score is **plain, valid sheet music** — the same notation any musician can read. To adapt it for a different venue, you bring along **a stack of sticky notes**: *"In Lisbon, change the tempo here. In Berlin, add this extra rest. In London, swap this instrument."* The base never changes; the sticky notes (called **overlays**) modify it for each performance.

This is the Kustomize philosophy: **plain YAML files + declarative patches**. No templating language, no `{{ }}`, no whitespace bugs from indentation in templates. Just YAML overlaying YAML.

In this lab, you'll restructure an application using a `base` directory and three overlays (`dev`, `staging`, `prod`), and you'll deploy each environment from the same base with different patches. By the end, you'll have a strong sense of when Kustomize wins (small to medium internal apps, GitOps repos, environment-only differences) and when Helm wins (distributable packages, complex conditional logic, third-party software).

---

## 🎯 Learning Objectives

- Understand Kustomize's overlay model and how it differs from Helm
- Structure an app as `base` + `overlays/{dev,staging,prod}`
- Use `kustomization.yaml` to compose resources
- Apply common patch types: strategic merge, JSON 6902, replacements
- Use `configMapGenerator` and `secretGenerator` for environment-specific config
- Use `images:` to override container image tags per environment
- Use `commonLabels` and `namePrefix` to differentiate resources
- Use `kubectl apply -k` (built-in) and the standalone `kustomize` CLI
- Know when to choose Kustomize over Helm (and vice versa)

---

## 📋 Prerequisites

| Item | Required | Check |
|------|----------|-------|
| AKS cluster | Yes | `kubectl get nodes` |
| `kubectl` 1.28+ | Yes | `kubectl version --client` (built-in `apply -k`) |
| `kustomize` standalone CLI | Optional | `kustomize version` (better for `kustomize build` previews) |

> 💡 Install standalone Kustomize: `brew install kustomize` or download from [the releases page](https://kubectl.docs.kubernetes.io/installation/kustomize/).

---

## 🧠 Key Concepts

**Base** — A directory containing a complete, deployable set of plain Kubernetes manifests. It must be valid on its own.

**Overlay** — A directory that references a base (or another overlay) and applies modifications. Overlays are also valid Kustomize roots.

**kustomization.yaml** — The manifest of a Kustomize directory. Lists which files/resources to include and what transformations to apply.

**Strategic merge patch** — A YAML patch that "merges" with the original. You only specify what changes; Kustomize understands the structure.

**JSON 6902 patch** — A more surgical patch format using operations (`add`, `replace`, `remove`) at JSON paths. Useful when strategic merge isn't precise enough.

**Generator** — Creates new resources from inputs. `configMapGenerator` and `secretGenerator` are the most common.

**Hash suffix** — Generated ConfigMaps/Secrets get a hash suffix in their name (e.g., `app-config-7c2b8`). When the data changes, the hash changes, and Pods auto-rollout. This solves the *"how do I make Pods notice ConfigMap changes?"* problem from Lab 3.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Set up the directory structure

```bash
mkdir -p ~/lab06/cloudlaunch/{base,overlays/dev,overlays/staging,overlays/prod}
cd ~/lab06/cloudlaunch
tree
```

### Step 2 — Build the base

The base is a fully functional, environment-agnostic deployment. Create the following files inside `base/`:

**`base/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudlaunch
  labels:
    app: cloudlaunch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudlaunch
  template:
    metadata:
      labels:
        app: cloudlaunch
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:1.0.0
          args:
            - "-text=Hello from base"
            - "-listen=:5678"
          ports:
            - containerPort: 5678
          envFrom:
            - configMapRef:
                name: app-config
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

**`base/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cloudlaunch
spec:
  selector:
    app: cloudlaunch
  ports:
    - port: 80
      targetPort: 5678
```

**`base/kustomization.yaml`:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=base
      - LOG_LEVEL=info

commonLabels:
  app.kubernetes.io/managed-by: kustomize
  app.kubernetes.io/part-of: cloudlaunch
```

**Preview what this renders:**

```bash
kubectl kustomize base/
# OR with the standalone CLI:
kustomize build base/
```

You'll see the Deployment, Service, and a generated ConfigMap with a hash suffix like `app-config-92h74cgf6f`.

### Step 3 — Build the dev overlay

Inside `overlays/dev/`, create:

**`overlays/dev/kustomization.yaml`:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: cloudlaunch-dev
namePrefix: dev-

resources:
  - ../../base

commonLabels:
  environment: dev

replicas:
  - name: cloudlaunch
    count: 1

images:
  - name: hashicorp/http-echo
    newTag: "1.0.0"

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - APP_ENV=dev
      - LOG_LEVEL=debug

patches:
  - target:
      kind: Deployment
      name: cloudlaunch
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/args
        value:
          - "-text=Hello from DEV"
          - "-listen=:5678"
```

**Preview:**

```bash
kubectl kustomize overlays/dev/
```

Look for:
- Resources renamed `dev-cloudlaunch`
- `namespace: cloudlaunch-dev` injected on every resource
- `environment: dev` label everywhere
- ConfigMap merged with dev-specific values
- Deployment args replaced with the dev message

### Step 4 — Build the staging overlay

**`overlays/staging/kustomization.yaml`:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: cloudlaunch-staging
namePrefix: stg-

resources:
  - ../../base

commonLabels:
  environment: staging

replicas:
  - name: cloudlaunch
    count: 2

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - APP_ENV=staging
      - LOG_LEVEL=info

patches:
  - target:
      kind: Deployment
      name: cloudlaunch
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/args
        value:
          - "-text=Hello from STAGING"
          - "-listen=:5678"
```

### Step 5 — Build the prod overlay (with a strategic merge patch)

This time, demonstrate the other patch style — a **strategic merge patch in a separate file**.

**`overlays/prod/deployment-patch.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudlaunch
spec:
  template:
    spec:
      containers:
        - name: app
          args:
            - "-text=Hello from PROD"
            - "-listen=:5678"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

**`overlays/prod/kustomization.yaml`:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: cloudlaunch-prod
namePrefix: prod-

resources:
  - ../../base

commonLabels:
  environment: prod

replicas:
  - name: cloudlaunch
    count: 4

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - APP_ENV=prod
      - LOG_LEVEL=warn

patches:
  - path: deployment-patch.yaml
    target:
      kind: Deployment
      name: cloudlaunch
```

**Preview:**

```bash
kubectl kustomize overlays/prod/
```

Notice how the patch only specified what changed (args, resources). Kustomize merged the rest from the base. **Compare to JSON 6902 in the dev overlay** — strategic merge is shorter for whole-block changes; JSON 6902 is more surgical for single-field changes.

### Step 6 — Deploy each environment

Create the namespaces and apply:

```bash
kubectl create namespace cloudlaunch-dev
kubectl create namespace cloudlaunch-staging
kubectl create namespace cloudlaunch-prod

kubectl apply -k overlays/dev/
kubectl apply -k overlays/staging/
kubectl apply -k overlays/prod/
```

**Verify each environment:**

```bash
kubectl get all -n cloudlaunch-dev
kubectl get all -n cloudlaunch-staging
kubectl get all -n cloudlaunch-prod

# Test the messages
kubectl run curl --rm -it --image=curlimages/curl --restart=Never -n cloudlaunch-dev -- \
  curl http://dev-cloudlaunch
# Expected: Hello from DEV

kubectl run curl --rm -it --image=curlimages/curl --restart=Never -n cloudlaunch-prod -- \
  curl http://prod-cloudlaunch
# Expected: Hello from PROD
```

Three environments, **one base**, three small overlay files. No templating language.

### Step 7 — Demonstrate the ConfigMap hash auto-rollout

Update the dev ConfigMap by editing `overlays/dev/kustomization.yaml`:

```yaml
configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - APP_ENV=dev
      - LOG_LEVEL=trace      # changed from debug
      - NEW_KEY=new_value    # added
```

```bash
kubectl apply -k overlays/dev/

# Watch what happens
kubectl get pods -n cloudlaunch-dev -w
```

You'll see Pods rolling out — even though you only changed a ConfigMap. **Why?** The generated ConfigMap got a new hash suffix (e.g., `app-config-9abc123` instead of `app-config-92h74cgf6f`). The Deployment's reference to it changed, which triggered a rollout. Press `Ctrl+C` when done.

> 🧠 **This is huge.** Kustomize solves one of Kubernetes's classic gotchas: changing a ConfigMap doesn't normally restart Pods. The hash trick makes ConfigMap changes behave like image tag changes — automatic, safe rollouts.

### Step 8 — Use a remote base

Kustomize bases don't have to be local. They can be Git repos. Create `~/lab06/external-test/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - github.com/kubernetes-sigs/kustomize/examples/multibases/dev/?ref=master

namePrefix: my-
```

```bash
kubectl kustomize ~/lab06/external-test/
```

You can pin to a specific commit, tag, or branch. **This is how teams share base manifests across multiple repos** without copy-pasting.

### Step 9 — Combining transformers and patches (advanced taste)

Add to `overlays/prod/kustomization.yaml`:

```yaml
labels:
  - pairs:
      cost-center: "platform"
      tier: "production"
    includeSelectors: false   # apply only to metadata, not selectors

commonAnnotations:
  contact: "platform-team@yourdomain.com"
  prometheus.io/scrape: "true"
```

```bash
kubectl apply -k overlays/prod/
kubectl get deployment -n cloudlaunch-prod -o yaml | grep -A 5 annotations
```

Annotations and labels propagate to every resource the kustomization manages.

### Step 10 — Helm vs Kustomize — a quick mental model

| Question | Helm wins | Kustomize wins |
|---|---|---|
| Is this a distributable package for others to install? | ✅ | |
| Lots of conditional logic (if/else)? | ✅ | |
| Third-party software (ingress-nginx, cert-manager)? | ✅ | |
| Pure environment overlays for *your* code? | | ✅ |
| GitOps repo (ArgoCD/Flux) where readability matters? | | ✅ |
| Want to avoid templating language? | | ✅ |
| Need release tracking and rollback via the tool? | ✅ | |

**The truth:** the best teams use both. Helm for installing third-party charts and packaging products for distribution. Kustomize for arranging *their own* apps across environments. They even compose — you can `helm template | kustomize build -` if you want.

---

## ✅ Validation Checklist

```bash
kustomize build overlays/dev/                  # Renders without error
kustomize build overlays/staging/              # Renders without error
kustomize build overlays/prod/                 # Renders without error
kubectl get deploy -n cloudlaunch-dev          # 1 replica
kubectl get deploy -n cloudlaunch-staging      # 2 replicas
kubectl get deploy -n cloudlaunch-prod         # 4 replicas
kubectl get cm -n cloudlaunch-dev              # ConfigMap with hash suffix
```

---

## 🧹 Cleanup

```bash
kubectl delete -k overlays/dev/
kubectl delete -k overlays/staging/
kubectl delete -k overlays/prod/

kubectl delete namespace cloudlaunch-dev cloudlaunch-staging cloudlaunch-prod
```

---

## 🎓 What You Actually Learned

- **Kustomize is "YAML on YAML."** No template language. The base is real Kubernetes YAML; overlays are real Kubernetes YAML; patches are also YAML. You can read every file in isolation.
- **Generators with hash suffixes solve the ConfigMap rollout problem** that bites people in plain Kubernetes and even in Helm.
- **Strategic merge vs JSON 6902** — strategic merge is more readable for "change these blocks"; JSON 6902 is precise for "set this exact field."
- **Overlays compose.** You can have `base` → `overlays/cloud-aks` → `overlays/cloud-aks/prod`. Each layer adds or overrides. Useful for multi-cloud or multi-region.
- **Bases can live in Git.** Teams share them by referencing repos.
- **Helm and Kustomize aren't enemies.** They solve overlapping but distinct problems. Most production stacks use both.
- **Built into kubectl.** `kubectl apply -k` works without installing anything else, which makes Kustomize trivial to adopt.

---

## 🤔 Reflection Questions

1. Why does the generated ConfigMap have a hash suffix? What problem does that solve?
2. When would you choose JSON 6902 over a strategic merge patch?
3. If you have `commonLabels` in both base and overlay, what wins?
4. Can you have an overlay of an overlay? Why or why not?
5. Helm has built-in rollback. How would you replicate that workflow with Kustomize?
6. What happens to the `resources` listed in `base/kustomization.yaml` if you reference the base from an overlay? Do they apply?

---

## 📚 Further Reading

- [Kustomize Documentation](https://kustomize.io/)
- [Kustomize Examples](https://github.com/kubernetes-sigs/kustomize/tree/master/examples)
- [Kustomize vs Helm](https://www.cncf.io/blog/2023/04/19/kustomize-vs-helm/)
- [Kustomize Reference (kubectl docs)](https://kubectl.docs.kubernetes.io/references/kustomize/)

---

## ➡️ What's Next

In **Lab 7**, we shift from "make it work" to "lock it down." You'll implement **RBAC** (who can do what to which resources), **Network Policies** (which Pods can talk to which Pods), and **Pod Security Standards** (preventing Pods from running as root or escaping the container sandbox). This is the lab where you stop being a hobbyist and start thinking like a platform engineer responsible for a multi-tenant cluster.

---

*Author: Prince Chime · Series: Kubernetes on AKS*
