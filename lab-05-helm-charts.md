# Lab 5: Helm Charts — Packaging Your First Application

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟡 Intermediate
> **Estimated Time:** 90–120 minutes
> **Prerequisite:** Labs 1–4 completed

---

## 🎼 ELI5 — What Are We Actually Doing?

Up to now, every YAML you wrote was for one specific performance — one orchestra, one venue, one date. If you wanted to perform the same symphony in a different city, you'd copy the entire score and manually rewrite the venue address, the conductor's name, the start time, and the seating capacity in dozens of places. Miss one, and the show breaks.

Real touring orchestras don't work this way. They have a **performance package** — the master score, the staging diagrams, the lighting cues — with blanks for the parts that change per venue. Hand the package to any concert hall along with a one-page "venue sheet" (city, capacity, date, conductor), and the local team can stage the entire production correctly.

That performance package is a **Helm chart**.

A Helm chart is your application's deployment blueprint, with placeholders (`{{ .Values.image.tag }}`, `{{ .Values.replicaCount }}`) that get filled in from a `values.yaml` file at install time. One chart, many environments. Same code, different `values.yaml` for dev, staging, and prod. This is how every serious team ships Kubernetes apps.

In this lab, you'll create a Helm chart from scratch, package your own app, install it with different values for different environments, upgrade it, and roll it back — all with `helm` commands.

---

## 🎯 Learning Objectives

- Understand the structure of a Helm chart
- Create a chart from scratch with `helm create`
- Use Go templates to parameterize Kubernetes manifests
- Use `values.yaml` and environment-specific overrides
- Install, upgrade, rollback, and uninstall a release
- Use `helm template` and `helm lint` for safe iteration
- Understand chart dependencies and reusable subcharts

---

## 📋 Prerequisites

| Item | Required | Check |
|------|----------|-------|
| AKS cluster | Yes | `kubectl get nodes` |
| Helm 3 | Yes | `helm version` |
| (Optional) ingress-nginx from Lab 4 | Recommended | If you want to expose the chart externally |

---

## 🧠 Key Concepts

**Chart** — A directory of files that describes a related set of Kubernetes resources.

**Release** — An *installation* of a chart in a cluster. You can install the same chart many times under different release names (`myapp-dev`, `myapp-prod`).

**values.yaml** — The default configuration for the chart. Overridden per release with `--values` files or `--set` flags.

**Templates** — YAML files with Go template directives like `{{ .Values.image.repository }}` that produce final manifests when rendered.

**Helpers (`_helpers.tpl`)** — A file of reusable template snippets (e.g., a function to compute the full app name).

**Chart.yaml** — Metadata: chart name, version, app version, description, dependencies.

**Hooks** — Special annotations on resources that make them run at specific lifecycle points (pre-install, post-upgrade, etc.). Useful for migrations and tests.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Scaffold a chart

Create a working directory and let Helm generate a starter chart:

```bash
mkdir -p ~/lab05 && cd ~/lab05
helm create cloudlaunch
tree cloudlaunch -L 2
```

You'll see:

```
cloudlaunch/
├── Chart.yaml
├── charts/              # subcharts (dependencies)
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl
│   ├── tests/
│   └── ...
└── values.yaml
```

Helm gave you a working starter — a Deployment, Service, Ingress, ServiceAccount, and a test, all parameterized.

### Step 2 — Customize Chart.yaml

Edit `cloudlaunch/Chart.yaml`:

```yaml
apiVersion: v2
name: cloudlaunch
description: A demo web app for the AKS labs series
type: application
version: 0.1.0          # chart version
appVersion: "1.0.0"     # the app version this chart deploys
maintainers:
  - name: Prince Chime
    url: https://youtube.com/@princeugochime
```

> 🧠 **Two versions, two purposes:** `version` is the chart's own version (bump when you change templates). `appVersion` is the version of the application image (bump when you release new code).

### Step 3 — Examine the templates

Open `cloudlaunch/templates/deployment.yaml`. You'll see things like:

```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

These `{{ }}` blocks are Go template expressions. They pull from:

- `.Values` → your `values.yaml`
- `.Chart` → your `Chart.yaml`
- `.Release` → release-specific data (name, namespace, revision)

### Step 4 — Edit values.yaml

Open `cloudlaunch/values.yaml` and set:

```yaml
replicaCount: 2

image:
  repository: hashicorp/http-echo
  pullPolicy: IfNotPresent
  tag: "1.0.0"

# http-echo expects -text="..." as args
extraArgs:
  - "-text=Hello from Cloudlaunch (default values)"
  - "-listen=:5678"

service:
  type: ClusterIP
  port: 80
  targetPort: 5678

ingress:
  enabled: false        # we'll enable this per environment
  className: nginx
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

### Step 5 — Adjust the deployment template for our app

The default `deployment.yaml` doesn't pass extra args or use our targetPort. Edit `cloudlaunch/templates/deployment.yaml` to wire those in. Find the `containers:` block and adjust it like so:

```yaml
containers:
  - name: {{ .Chart.Name }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
    imagePullPolicy: {{ .Values.image.pullPolicy }}
    {{- with .Values.extraArgs }}
    args:
      {{- toYaml . | nindent 12 }}
    {{- end }}
    ports:
      - name: http
        containerPort: {{ .Values.service.targetPort }}
        protocol: TCP
    resources:
      {{- toYaml .Values.resources | nindent 12 }}
```

Also update the Service template if needed so `targetPort` uses `.Values.service.targetPort`. The starter chart usually uses the named port `http`, which works because we named our containerPort `http` above.

### Step 6 — Lint and dry-run before installing

**Lint** checks for syntax and best-practice issues:

```bash
helm lint ./cloudlaunch
```

**Template** renders the chart locally so you can inspect what will be applied — without touching the cluster:

```bash
helm template demo ./cloudlaunch | head -80
```

> 🧠 **Pro tip:** `helm template` is your best friend when debugging chart logic. It's faster than installing and seeing what breaks.

### Step 7 — Install the chart (dev environment)

Create a namespace and install:

```bash
kubectl create namespace cloudlaunch-dev

helm install cloudlaunch-dev ./cloudlaunch \
  --namespace cloudlaunch-dev
```

**Verify the release:**

```bash
helm list -n cloudlaunch-dev
kubectl get all -n cloudlaunch-dev
```

You'll see `STATUS: deployed`. The chart created a Deployment, Service, ServiceAccount, and a Pod test resource — all from your one chart.

### Step 8 — Create environment-specific values files

This is the whole point of Helm. Same chart, different environments.

Create `values-dev.yaml`:

```yaml
replicaCount: 1
extraArgs:
  - "-text=Hello from DEV"
  - "-listen=:5678"
resources:
  requests:
    cpu: 25m
    memory: 32Mi
  limits:
    cpu: 100m
    memory: 64Mi
```

Create `values-prod.yaml`:

```yaml
replicaCount: 4
extraArgs:
  - "-text=Hello from PROD"
  - "-listen=:5678"
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: cloudlaunch.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
```

**Upgrade the dev release with the new values:**

```bash
helm upgrade cloudlaunch-dev ./cloudlaunch \
  --namespace cloudlaunch-dev \
  --values values-dev.yaml
```

**Install a separate prod release:**

```bash
kubectl create namespace cloudlaunch-prod

helm install cloudlaunch-prod ./cloudlaunch \
  --namespace cloudlaunch-prod \
  --values values-prod.yaml
```

Two completely independent releases of the same chart, each with appropriate sizing.

```bash
helm list --all-namespaces
kubectl get pods -n cloudlaunch-dev
kubectl get pods -n cloudlaunch-prod
```

### Step 9 — Use --set for one-off overrides

Sometimes you want to override a single value without editing a file:

```bash
helm upgrade cloudlaunch-dev ./cloudlaunch \
  --namespace cloudlaunch-dev \
  --values values-dev.yaml \
  --set replicaCount=3
```

> ⚠️ `--set` values are not stored in any file — they live only in the release. For anything you'll want to repeat, prefer values files.

### Step 10 — Practice upgrades and rollbacks

**See release history:**

```bash
helm history cloudlaunch-dev -n cloudlaunch-dev
```

You'll see multiple revisions. Each `upgrade` creates a new revision.

**Make a deliberately bad change** — edit `values-dev.yaml`:

```yaml
image:
  tag: "this-tag-does-not-exist"
```

```bash
helm upgrade cloudlaunch-dev ./cloudlaunch \
  --namespace cloudlaunch-dev \
  --values values-dev.yaml

kubectl get pods -n cloudlaunch-dev
# You'll see ImagePullBackOff
```

**Roll back to the previous revision:**

```bash
helm rollback cloudlaunch-dev -n cloudlaunch-dev
# Or specify a revision number:
helm rollback cloudlaunch-dev 2 -n cloudlaunch-dev

helm history cloudlaunch-dev -n cloudlaunch-dev
kubectl get pods -n cloudlaunch-dev
```

Notice the rollback also creates a new revision — Helm never erases history.

### Step 11 — Add a custom helper template

Open `cloudlaunch/templates/_helpers.tpl`. The starter already includes helpers for `cloudlaunch.fullname`, `cloudlaunch.labels`, etc. Add a custom one:

```gotemplate
{{/*
Custom helper: returns "dev" / "prod" / "unknown" based on release name suffix.
*/}}
{{- define "cloudlaunch.environment" -}}
{{- if hasSuffix "-dev" .Release.Name -}}
dev
{{- else if hasSuffix "-prod" .Release.Name -}}
prod
{{- else -}}
unknown
{{- end -}}
{{- end -}}
```

Use it in `templates/deployment.yaml`, adding a label:

```yaml
metadata:
  labels:
    {{- include "cloudlaunch.labels" . | nindent 4 }}
    environment: {{ include "cloudlaunch.environment" . }}
```

```bash
helm upgrade cloudlaunch-dev ./cloudlaunch \
  --namespace cloudlaunch-dev \
  --values values-dev.yaml

kubectl get deployment -n cloudlaunch-dev --show-labels
```

### Step 12 — Run the chart's built-in test

Helm charts can include test Pods that validate the deployment:

```bash
helm test cloudlaunch-dev -n cloudlaunch-dev
```

The starter chart's test Pod just runs `wget` against the Service. Useful as a smoke test in CI/CD.

### Step 13 — Package and (optionally) publish

Package the chart into a versioned `.tgz`:

```bash
helm package ./cloudlaunch
ls cloudlaunch-0.1.0.tgz
```

This is what you'd push to a chart repository (Azure Container Registry supports OCI Helm charts, GitHub Pages with chart-releaser, ChartMuseum, Harbor, etc.) so other teams can `helm install` it.

**Push to ACR (if you have one):**

```bash
ACR_NAME=youracrname
helm registry login ${ACR_NAME}.azurecr.io \
  --username $(az acr credential show -n $ACR_NAME --query username -o tsv) \
  --password $(az acr credential show -n $ACR_NAME --query passwords[0].value -o tsv)

helm push cloudlaunch-0.1.0.tgz oci://${ACR_NAME}.azurecr.io/helm
```

Now anyone with access can `helm install myrelease oci://${ACR_NAME}.azurecr.io/helm/cloudlaunch --version 0.1.0`.

---

## ✅ Validation Checklist

```bash
helm list --all-namespaces                      # Both releases listed
kubectl get pods -n cloudlaunch-dev             # Running
kubectl get pods -n cloudlaunch-prod            # Running, with prod replica count
helm history cloudlaunch-dev -n cloudlaunch-dev # Multiple revisions
helm template ./cloudlaunch --values values-prod.yaml | grep "Hello from PROD"
```

---

## 🧹 Cleanup

```bash
helm uninstall cloudlaunch-dev -n cloudlaunch-dev
helm uninstall cloudlaunch-prod -n cloudlaunch-prod
kubectl delete namespace cloudlaunch-dev cloudlaunch-prod
```

---

## 🎓 What You Actually Learned

- **A chart is a parameterized package**, not just YAML. The same chart deploys to any environment by swapping values files.
- **Releases are independent installations.** You can run `cloudlaunch-dev` and `cloudlaunch-prod` side by side; they don't interfere.
- **Helm tracks revisions automatically.** Every install/upgrade is a versioned snapshot you can roll back to. This is `kubectl rollout undo` on steroids.
- **`helm template` is for debugging; `helm install --dry-run` is for cluster-side validation.** Use both.
- **Helpers reduce duplication.** `_helpers.tpl` lets you DRY up label blocks, naming logic, and conditionals.
- **Charts are distributable.** Package once, push to a registry, install anywhere. This is how Bitnami, prometheus-community, ingress-nginx, and every other open-source project ships to Kubernetes.
- **Helm is not Kubernetes.** It's a client-side templating + release-management tool. The cluster sees the rendered YAML; Helm tracks the metadata in a Secret in your release namespace.

---

## 🤔 Reflection Questions

1. What does Helm store in the cluster to track a release? Where does it live?
2. If you uninstall a release and reinstall it, do you keep your revision history?
3. What's the difference between `helm upgrade --install` and `helm install`?
4. Why is `helm template` useful even when you have a working cluster?
5. When would you use a subchart vs separate independent charts?
6. What happens if two releases of the same chart write to the same `ConfigMap` name?

---

## 📚 Further Reading

- [Helm Documentation](https://helm.sh/docs/)
- [Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [The Chart Template Developer's Guide](https://helm.sh/docs/chart_template_guide/)
- [Helm + ACR (OCI artifacts)](https://learn.microsoft.com/en-us/azure/aks/quickstart-helm)

---

## ➡️ What's Next

Helm is powerful, but it has critics — Go templating produces YAML, and YAML-from-templates can get hairy fast (whitespace bugs, complex conditionals, unreadable diffs). In **Lab 6**, we'll explore the alternative: **Kustomize**. Instead of templates, Kustomize uses overlays — *"take this base and patch it with these dev/staging/prod-specific changes."* No string templating, just YAML on YAML. You'll see when each tool wins, and you'll come away with both in your toolkit.

---

*Author: Prince Chime · Series: Kubernetes on AKS*
