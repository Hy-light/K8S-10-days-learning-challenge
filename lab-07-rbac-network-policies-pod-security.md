# Lab 7: RBAC, Network Policies & Pod Security

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟠 Advanced
> **Estimated Time:** 90–120 minutes
> **Prerequisite:** Labs 1–6 completed

---

## 🎼 ELI5 — What Are We Actually Doing?

Up to now, your concert hall has been wide open. The conductor can do anything. Any musician can walk into any room. Any guest can wander into the dressing rooms. That's fine for a single rehearsal, but a real venue running multiple shows for multiple paying clients needs **three layers of security**:

1. **Backstage passes (RBAC)** — Different people get different access. The conductor can do anything; a guest violinist can only enter the strings room; a janitor can only enter the maintenance corridor. The pass system answers *"who is allowed to do what to which resources?"*

2. **Hallway access rules (Network Policies)** — Even with valid passes, you don't want random communication between groups. The brass section doesn't need to walk through the strings dressing room. By default in Kubernetes, **every Pod can talk to every other Pod**. That's a security hole the size of the building. Network Policies are the locked doors and keycards that say *"only these specific connections are allowed."*

3. **What musicians can bring on stage (Pod Security)** — Some musicians might try to bring fireworks, or hijack the lighting rig, or run with scissors. The venue has rules: *"no flammables, no privileged equipment, no running as root."* Pod Security Standards enforce these at the cluster door — Pods that violate the policy are simply not admitted.

This lab transforms your cluster from "open campus" to "secure venue." The same patterns apply whether you're running one team's apps or a multi-tenant platform serving the whole company.

---

## 🎯 Learning Objectives

- Understand the Subject → Role → Resource model of Kubernetes RBAC
- Create Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings
- Use ServiceAccounts to grant Pods their own permissions
- Test access with `kubectl auth can-i`
- Implement default-deny Network Policies and selectively allow traffic
- Apply Pod Security Standards (Baseline, Restricted) at the namespace level
- Integrate with Azure AD (Entra ID) for human user authentication on AKS
- Briefly preview Workload Identity for Pods accessing Azure services

---

## 📋 Prerequisites

| Item | Required | Check |
|------|----------|-------|
| AKS cluster | Yes | `kubectl get nodes` |
| AKS cluster has Network Policy enabled | Yes | See Step 0 below |
| Cluster admin access | Yes | `kubectl auth can-i '*' '*' --all-namespaces` should be `yes` |

### Step 0 — Enable Network Policy on your AKS cluster

Network policies require a CNI that supports them. AKS supports **Azure CNI Overlay with Cilium** (recommended) or **Calico**.

If your Lab 1 cluster doesn't have a network policy engine, **you'll need to recreate it**. AKS doesn't allow enabling network policy on an existing cluster.

```bash
# Recreate with Cilium-based dataplane
az group delete --name rg-aks-lab01 --yes
az group create --name rg-aks-lab07 --location westeurope

az aks create \
  --resource-group rg-aks-lab07 \
  --name aks-lab07 \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --pod-cidr 10.244.0.0/16 \
  --enable-managed-identity \
  --generate-ssh-keys

az aks get-credentials -g rg-aks-lab07 -n aks-lab07
```

> 💡 **Why Cilium?** It's eBPF-based, very fast, and the modern default. Calico is also supported (`--network-policy calico`) and is widely used.

---

## 🧠 Key Concepts

**ServiceAccount** — An identity for *Pods*. Every Pod runs as one (`default` if you don't specify).

**User** — An identity for *humans*. Kubernetes doesn't manage users itself; it delegates to an external identity provider (Azure AD on AKS).

**Role / ClusterRole** — A collection of permissions. *"Can list Pods, can create Deployments."* A Role is namespace-scoped; a ClusterRole is cluster-wide.

**RoleBinding / ClusterRoleBinding** — Connects a Subject (user, group, or ServiceAccount) to a Role. *"User Alice gets the developer Role in the `frontend` namespace."*

**Network Policy** — A spec describing allowed ingress and egress for Pods that match a label selector. **They are deny-by-default once any policy targets a Pod.** Until then, all traffic is allowed.

**Pod Security Standards (PSS)** — Three levels: `privileged` (no restrictions), `baseline` (no obvious bad practices), `restricted` (heavily hardened). Applied per namespace via labels.

**Workload Identity** — Lets a Pod authenticate to Azure as a Managed Identity, without storing credentials. We'll preview this; full implementation is its own deep topic.

---

## 🚀 Step-by-Step Instructions

### Step 1 — Set up two namespaces with different security profiles

```bash
# Namespace where developers can deploy freely
kubectl create namespace team-app

# Namespace with strict security
kubectl create namespace team-secure
```

Apply Pod Security Standards as namespace labels:

```bash
# Baseline: no privileged containers, no host networking, etc.
kubectl label namespace team-app \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest

# Restricted: stricter (no root, must drop capabilities, must seccomp, etc.)
kubectl label namespace team-secure \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

### Step 2 — Test that Pod Security Standards are enforced

Try to deploy a Pod that runs as root in `team-secure`:

```yaml
# bad-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
    - name: app
      image: nginx:1.27-alpine
      # No securityContext specified — runs as root by default
```

```bash
kubectl apply -f bad-pod.yaml -n team-secure
```

You'll get a clear admission error explaining what's missing (must run as non-root, must drop ALL capabilities, must set seccomp profile, etc.). **The Pod is rejected at the cluster door** — it never even starts.

Now create a compliant version, `good-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginxinc/nginx-unprivileged:1.27-alpine
      ports:
        - containerPort: 8080
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: nginx-cache
          mountPath: /var/cache/nginx
        - name: nginx-run
          mountPath: /var/run
  volumes:
    - name: tmp
      emptyDir: {}
    - name: nginx-cache
      emptyDir: {}
    - name: nginx-run
      emptyDir: {}
```

```bash
kubectl apply -f good-pod.yaml -n team-secure
kubectl get pod good-pod -n team-secure
```

This one runs. Notice we used `nginx-unprivileged` (designed to run as a non-root user) and explicitly dropped capabilities. **This is what production-grade Pod specs look like.**

### Step 3 — Create a ServiceAccount with limited permissions

A common case: a Pod needs to read its own namespace's ConfigMaps but nothing else.

Create `sa-and-role.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader
  namespace: team-app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: team-app
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-binding
  namespace: team-app
subjects:
  - kind: ServiceAccount
    name: config-reader
    namespace: team-app
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f sa-and-role.yaml
```

### Step 4 — Run a Pod as that ServiceAccount and test

```yaml
# reader-pod.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config
  namespace: team-app
data:
  greeting: "Hello, RBAC"
---
apiVersion: v1
kind: Pod
metadata:
  name: reader-pod
  namespace: team-app
spec:
  serviceAccountName: config-reader
  containers:
    - name: kubectl
      image: bitnami/kubectl:latest # just to keep things simple
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f reader-pod.yaml

# What can this ServiceAccount do?
kubectl exec -n team-app reader-pod -- kubectl auth can-i list configmaps
# yes
kubectl exec -n team-app reader-pod -- kubectl auth can-i list pods
# no
kubectl exec -n team-app reader-pod -- kubectl auth can-i list configmaps -n kube-system
# no — the Role is namespace-scoped

# Read the ConfigMap
kubectl exec -n team-app reader-pod -- kubectl get configmap example-config -o yaml
```

The Pod can read ConfigMaps in its own namespace and nothing else. **Principle of least privilege in action.**

### Step 5 — Default-deny Network Policy

Let's lock down `team-secure`. Create `default-deny.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-secure
spec:
  podSelector: {}        # selects ALL pods in this namespace
  policyTypes:
    - Ingress
    - Egress
```

Note there are no `ingress:` or `egress:` rules — meaning *nothing* is allowed.

```bash
kubectl apply -f default-deny.yaml
```

**Test that it works.** Run a curl Pod and try to reach DNS or the internet:

```bash
kubectl run net-test --image=curlimages/curl --restart=Never -n team-secure -- sleep 3600

# Wait, but this Pod won't satisfy restricted PSS. Let's use the good-pod approach.
```

Apply this hardened test Pod, `net-test.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: net-test
  namespace: team-secure
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: c
      image: curlimages/curl:8.6.0
      command: ["sleep", "3600"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: { drop: ["ALL"] }
```

```bash
kubectl apply -f net-test.yaml

# Try to reach the internet — this will hang/fail
kubectl exec -n team-secure net-test -- curl --max-time 5 https://example.com
# (will time out)

# Try DNS — also blocked
kubectl exec -n team-secure net-test -- nslookup kubernetes.default
```

The default-deny policy is in effect.

### Step 6 — Selectively allow traffic

Two things almost every Pod needs:

1. **DNS** — to resolve cluster service names
2. **Specific app communication** — only what's required

Create `allow-dns.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: team-secure
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl apply -f allow-dns.yaml

# DNS now works
kubectl exec -n team-secure net-test -- nslookup kubernetes.default
# OR 
kubectl exec -n team-secure net-test -- nslookup kubernetes.default.svc.cluster.local
```

Now allow this namespace's Pods to reach a specific external API only. Let's say you have an internal service in `team-app` you want to reach:

Deploy a target in `team-app`:

```bash
kubectl create deployment echo --image=nginx:alpine -n team-app
kubectl expose deployment echo --port=80 --target-port=80 -n team-app
```

Allow `team-secure` to reach `team-app/echo`:

```yaml
# allow-team-app-echo.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-team-app-echo
  namespace: team-secure
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: team-app
          podSelector:
            matchLabels:
              app: echo
      ports:
        - protocol: TCP
          port: 80
```

```bash
# Make sure team-app has the right namespace label (auto-added in modern K8s)
kubectl label namespace team-app kubernetes.io/metadata.name=team-app --overwrite

kubectl apply -f allow-team-app-echo.yaml

# Test - this should work
kubectl exec -n team-secure net-test -- curl -s http://echo.team-app.svc.cluster.local
```

But the target side also needs to allow it! Add an ingress rule on the `team-app` side:

```yaml
# allow-from-secure.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-team-secure
  namespace: team-app
spec:
  podSelector:
    matchLabels:
      app: echo
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: team-secure
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl apply -f allow-from-secure.yaml
```

> 🧠 **The big idea:** Network policies are **additive**. Once any policy targets a Pod, that Pod is in deny-by-default mode. Each policy adds permitted traffic. Both ends (egress on source, ingress on destination) need to allow the connection.

### Step 7 — Human authentication via Azure AD (Entra ID)

On AKS, you don't manage human users in Kubernetes. You use Azure AD groups and bind them to Roles.

> 💡 **This step requires Azure AD admin or equivalent rights** in your tenant. If you don't have that, read through and skip the actual commands.

Enable Azure AD integration on your cluster (if you didn't at create time):

```bash
# Create the group if you have not
az ad group create --display-name "AKS Developers" --mail-nickname "AKSDevelopers"

# Get your Azure AD group object IDs
AAD_GROUP=$(az ad group list --display-name "AKS Developers" --query "[].id" -o tsv)

# Enable AAD integration with that group as cluster admins
az aks update -g rg-aks-lab07 -n aks-lab07 \
  --enable-aad \
  --aad-admin-group-object-ids $AAD_GROUP
```

Now bind an Azure AD group to a namespace-scoped Role:

```yaml
# aad-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-can-edit
  namespace: team-app
subjects:
  - kind: Group
    name: "<AAD-group-object-id>"  # Object ID, not display name
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit                       # built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

When a member of that AAD group runs `az aks get-credentials --resource-group rg-aks-lab07 --name aks-lab07`, they'll authenticate with their Azure AD identity and get the `edit` permissions in `team-app` only.

You can also give the group a read-only permission on the cluster scope. The last apply makes it possible for them to only read from a single namespace. Check on the portal to verify this.

```yaml
# aad-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: aad-developers-view
subjects:
  - kind: Group
    name: "<AAD-group-object-id>"   # Replace with your actual group object ID
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view                         # built-in ClusterRole (read-only across all namespaces)
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f aad-clusterrolebinding.yaml
```

### Step 8 — Workload Identity preview

Pods that need Azure resources (e.g., read from a Storage Account or Key Vault) shouldn't store credentials. **Workload Identity** maps a Kubernetes ServiceAccount to an Azure Managed Identity.

The flow looks like this:

```bash
# Enable Workload Identity on the cluster
az aks update -g rg-aks-lab07 -n aks-lab07 \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get the OIDC issuer URL
OIDC_ISSUER=$(az aks show -g rg-aks-lab07 -n aks-lab07 --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Create a User-Assigned Managed Identity
az identity create -g rg-aks-lab07 -n my-app-identity
CLIENT_ID=$(az identity show -g rg-aks-lab07 -n my-app-identity --query clientId -o tsv)

# Create the federated credential mapping a K8s SA to the identity
az identity federated-credential create \
  -n my-app-fed-cred \
  -g rg-aks-lab07 \
  --identity-name my-app-identity \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:team-app:azure-app" \
  --audience "api://AzureADTokenExchange"
```

Then annotate the ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-app
  namespace: team-app
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID>"
```

And label the Pod:

```yaml
metadata:
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: azure-app
```

The Pod can now use Azure SDKs with `DefaultAzureCredential` and authenticate as the Managed Identity — **no secrets stored anywhere**. This is the modern, recommended pattern, and it replaces the older AAD Pod Identity (deprecated).

> 💡 We're keeping this short on purpose — Workload Identity is its own deep topic. The point is: by Lab 7, you should know it exists and what problem it solves.

---

## ✅ Validation Checklist

```bash
kubectl get ns team-app team-secure --show-labels    # PSS labels visible
kubectl auth can-i list configmaps -n team-app \
  --as=system:serviceaccount:team-app:config-reader  # yes
kubectl auth can-i list pods -n team-app \
  --as=system:serviceaccount:team-app:config-reader  # no
kubectl get networkpolicy -n team-secure             # default-deny + allows
kubectl exec -n team-secure net-test -- nslookup kubernetes.default  # works
kubectl exec -n team-secure net-test -- curl --max-time 5 https://example.com  # times out
```

---

## 🧹 Cleanup

```bash
kubectl delete namespace team-app team-secure

# If you created a fresh cluster for this lab and don't need it for the next ones:
az group delete --name rg-aks-lab07 --yes --no-wait
```

---

## 🎓 What You Actually Learned

- **Authorization in Kubernetes is a graph.** Subject (user/group/SA) → Binding → Role → Resources/Verbs. Master that mental model and the YAML is just plumbing.
- **ServiceAccounts are how Pods get identity.** Every Pod runs as one. Default is `default`, which has minimal permissions but is still a real identity.
- **`can-i` is your debugging superpower.** Use `kubectl auth can-i ... --as=...` to test permissions without actually attempting actions.
- **Network policies are deny-by-default *once applied*.** Without any policy, all traffic is allowed. With one policy targeting a Pod, only what's explicitly allowed gets through.
- **Both sides need to agree.** Egress on the source's side AND ingress on the destination's side must both permit a connection.
- **Pod Security Standards live in the admission layer.** Violating Pods are rejected before they exist. This is far more powerful than catching issues at runtime.
- **AKS hands off identity to Azure AD.** Don't manage human users inside Kubernetes; let Azure AD do it and bind groups to Roles.
- **Workload Identity replaces stored credentials.** No more secrets in Pods reaching Azure services.
- **Defense in depth.** RBAC (who) + Network Policy (where) + PSS (what) + Workload Identity (how authenticated) — none alone is enough; together they're strong.

---

## 🤔 Reflection Questions

1. What's the difference between a Role and a ClusterRole? When does it matter?
2. If a Pod has no explicit `serviceAccountName`, what identity does it use? What can that identity do?
3. Why do network policies require a CNI that supports them? What's actually enforcing the rules?
4. The `restricted` PSS rejected `nginx:1.27-alpine`. Why? What specifically did it object to?
5. What's the security risk of using base64 Secrets without encryption-at-rest, even if RBAC restricts access?
6. Workload Identity uses OIDC token exchange. What does the federation actually establish?

---

## 📚 Further Reading

- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Network Policy Recipes (must-read)](https://github.com/ahmetb/kubernetes-network-policy-recipes)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [AKS Azure AD Integration](https://learn.microsoft.com/en-us/azure/aks/azure-ad-rbac)
- [Workload Identity on AKS](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)

---

## ➡️ What's Next

In **Lab 8**, we tackle scale. Apps don't need the same number of Pods at 3 a.m. as at 3 p.m. You'll set up the **Horizontal Pod Autoscaler (HPA)** to scale on CPU and memory, then graduate to **KEDA** — the Kubernetes Event-Driven Autoscaler — which scales on richer signals like queue depth, Service Bus messages, Prometheus metrics, and even cron schedules. By the end, your apps will breathe with traffic instead of being statically over-provisioned.

---

*Author: Prince Chime · Series: Kubernetes on AKS*
