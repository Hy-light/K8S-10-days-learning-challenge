# Lab 4: Ingress Controllers & TLS with cert-manager

> **Series:** Kubernetes on AKS — From Basics to Enterprise
> **Level:** 🟡 Intermediate
> **Estimated Time:** 90–120 minutes
> **Prerequisite:** Labs 1–3 completed, plus a domain name you control (e.g., `yourdomain.com`)

---

## 🎼 ELI5 — What Are We Actually Doing?

In Lab 2, every public-facing app required its own `LoadBalancer` Service — its own dedicated stage door, with its own dedicated security guard, on its own dedicated street address. That works for one orchestra, but imagine running a music festival with 30 ensembles. Thirty stage doors? Thirty security guards? Thirty addresses to memorize? Absurd.

Real venues solve this with **a single grand entrance and a smart usher**. The usher reads each guest's ticket — *"Concert Hall A? Down the left corridor. Jazz Lounge? Up the stairs."* — and routes them to the correct room. One door, many destinations.

That smart usher is an **Ingress Controller**. It's a single front door for HTTP/HTTPS traffic that reads each request's hostname and path (e.g., `api.yourdomain.com/users` vs `app.yourdomain.com/login`) and routes to the correct internal Service.

We'll also tackle the next problem: every modern venue requires an ID check at the door — and so does every modern website. Browsers warn users away from anything that isn't HTTPS. Manually buying and rotating TLS certificates is tedious. **cert-manager** is a robot that automatically requests, installs, and renews free Let's Encrypt certificates whenever you ask for one. No manual work, no expired certs at 3 a.m.

By the end of this lab, your apps will be reachable at `https://app1.yourdomain.com` and `https://app2.yourdomain.com` through a single load balancer, with valid TLS certificates that auto-renew forever.

---

## 🎯 Learning Objectives

- Install the NGINX Ingress Controller via Helm
- Understand the relationship between Ingress, IngressClass, and the controller
- Route traffic to multiple Services based on hostname and path
- Install cert-manager and configure a Let's Encrypt ClusterIssuer
- Obtain automatic TLS certificates for your Ingress hosts
- Understand HTTP-01 vs DNS-01 challenges (and which one we'll use)

---

## 📋 Prerequisites

| Item | Required | Notes |
|------|----------|-------|
| AKS cluster | Yes | From Lab 1 |
| Helm 3 | Yes | `helm version` (install via `brew install helm` or [official guide](https://helm.sh/docs/intro/install/)) |
| A real domain | Yes | You need to point DNS at the Ingress public IP |
| DNS provider access | Yes | Cloudflare, Azure DNS, GoDaddy, Namecheap — anywhere you can edit A records. I do recommend Cloudflare because it was very easy to set-up and affordable |


> 💡 **No domain?** You can skip cert-manager and test Ingress with `nip.io` (a free wildcard DNS service that maps any IP to a hostname). Example: `1.2.3.4.nip.io` resolves to `1.2.3.4`. TLS won't work properly with nip.io, but routing will.

---

## 🧠 Key Concepts

**Ingress Controller** — The actual running software (NGINX, Traefik, HAProxy, Azure Application Gateway, etc.) that listens on ports 80/443 and routes traffic. You install it once.

**Ingress (resource)** — A YAML object describing routing rules. *"Send `app1.example.com` to the `app1-svc` Service on port 80."* You create one per app.

**IngressClass** — A label that connects an Ingress resource to a specific Ingress Controller (in case you have multiple — e.g., one for internal traffic, one for external).

**cert-manager** — A Kubernetes operator that automates TLS certificate lifecycles. It watches for special annotations on Ingress resources and provisions certs accordingly.

**ClusterIssuer** — A cluster-wide cert-manager configuration that says "use Let's Encrypt with these settings to issue certificates."

**HTTP-01 challenge** — Let's Encrypt's way of verifying you own a domain: it makes an HTTP request to `http://yourdomain.com/.well-known/acme-challenge/...` and expects a specific response. **This is why your domain MUST be pointing to your Ingress IP before requesting a cert.**

---

## 🚀 Step-by-Step Instructions

### Step 1 — Set up namespaces

```bash
kubectl create namespace ingress-nginx
kubectl create namespace cert-manager
kubectl create namespace lab04
kubectl config set-context --current --namespace=lab04
```

### Step 2 — Install NGINX Ingress Controller via Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.service.externalTrafficPolicy=Local
```

**What that flag does:**

- `externalTrafficPolicy=Local` → preserves the real client IP. Without this, your apps see the node's IP instead of the actual visitor's IP.

**Wait for the public IP:**

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller --watch
```

When `EXTERNAL-IP` is assigned, save it:

```bash
INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Ingress IP: $INGRESS_IP"
```

### Step 3 — Point your DNS to the Ingress IP

In your DNS provider, create two **A records** pointing at `$INGRESS_IP`:

```
app1.yourdomain.com    A    <INGRESS_IP>
app2.yourdomain.com    A    <INGRESS_IP>
```

Wait for propagation:

```bash
nslookup app1.yourdomain.com
# Should return $INGRESS_IP
```

> ⚠️ **Don't skip this.** The Let's Encrypt cert request will fail later if DNS isn't resolving correctly.

### Step 4 — Deploy two sample applications

Create `apps.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels: { app: app1 }
  template:
    metadata:
      labels: { app: app1 }
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:1.0.0
          args: ["-text=Hello from APP 1"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app1-svc
spec:
  selector: { app: app1 }
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 2
  selector:
    matchLabels: { app: app2 }
  template:
    metadata:
      labels: { app: app2 }
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:1.0.0
          args: ["-text=Hello from APP 2"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app2-svc
spec:
  selector: { app: app2 }
  ports:
    - port: 80
      targetPort: 5678
```

```bash
kubectl apply -f apps.yaml
kubectl get pods,svc
```

### Step 5 — Create your first Ingress (HTTP only, no TLS yet)

Create `ingress-http.yaml` (replace `yourdomain.com`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab04-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app1.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-svc
                port:
                  number: 80
    - host: app2.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2-svc
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-http.yaml
kubectl get ingress
```

**Test:**

```bash
curl http://app1.yourdomain.com
# Hello from APP 1
curl http://app2.yourdomain.com
# Hello from APP 2
```

🎉 **One load balancer, two apps, smart routing.** You've replaced two `LoadBalancer` Services with one Ingress.

### Step 6 — Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.15.0 \
  --set crds.enabled=true \
  --set replicaCount=2

kubectl get pods -n cert-manager
```

You should see three running pods: `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook`.

### Step 7 — Create a ClusterIssuer for Let's Encrypt

Create `cluster-issuer.yaml` (replace the email):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-real-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

```bash
kubectl apply -f cluster-issuer.yaml
kubectl describe clusterissuer letsencrypt-prod
```

The status should say `Ready: True`. If not, check the events.

> 🧪 **Tip:** Let's Encrypt's production endpoint has strict rate limits. While testing, switch the `server:` URL to the staging endpoint `https://acme-staging-v02.api.letsencrypt.org/directory`. Staging issues fake certs (browsers won't trust them), but you can iterate without burning your real-cert quota.

### Step 8 — Add TLS to your Ingress

Update your Ingress to request a certificate. Create `ingress-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab04-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app1.yourdomain.com
        - app2.yourdomain.com
      secretName: lab04-tls
  rules:
    - host: app1.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-svc
                port:
                  number: 80
    - host: app2.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2-svc
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-tls.yaml
```

**Watch the certificate flow happen:**

```bash
# A Certificate resource appears
kubectl get certificate

# A CertificateRequest is created
kubectl get certificaterequest

# An Order is placed with Let's Encrypt
kubectl get order

# Challenges are created and solved
kubectl get challenge

# Eventually, the Certificate becomes Ready
kubectl describe certificate lab04-tls
```

This whole dance usually takes 30–90 seconds. When the Certificate's `READY` column shows `True`, you have a valid TLS cert.

**Test it:**

```bash
curl https://app1.yourdomain.com
curl https://app2.yourdomain.com
```

Or open in a browser — you should see the padlock icon. **Free, valid, auto-renewing TLS for both apps.** 🔒

### Step 9 — Add path-based routing (bonus pattern)

You can also route by path, not just hostname. Add a new path under one host:

```yaml
- host: app1.yourdomain.com
  http:
    paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: app1-svc
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: app2-svc
            port:
              number: 80
```

This sends `app1.yourdomain.com/v1/*` to app1 and `app1.yourdomain.com/v2/*` to app2. Useful for API versioning or BFF patterns.

---

## ✅ Validation Checklist

```bash
kubectl get pods -n ingress-nginx          # Controller running
kubectl get svc -n ingress-nginx           # External IP assigned
kubectl get pods -n cert-manager           # 3 cert-manager pods Running
kubectl get clusterissuer                  # letsencrypt-prod Ready
kubectl get certificate                    # lab04-tls Ready True
curl -I https://app1.yourdomain.com        # HTTP/2 200, valid cert
```

---

## 🧹 Cleanup

```bash
# Lab resources
kubectl delete namespace lab04

# Ingress controller and cert-manager (if you don't need them for next labs)
helm uninstall ingress-nginx -n ingress-nginx
helm uninstall cert-manager -n cert-manager
kubectl delete namespace ingress-nginx cert-manager

# Don't forget to remove the DNS A records you created
```

> 💡 **If continuing to Lab 5**, keep ingress-nginx and cert-manager installed — you'll reuse them.

---

## 🎓 What You Actually Learned

- **One Ingress > many LoadBalancers.** A single load balancer with smart routing is cheaper, easier to manage, and the standard production pattern.
- **The IngressClass connects rules to a controller.** This lets you run multiple Ingress controllers (e.g., one internal, one external) without conflicts.
- **cert-manager is a Kubernetes operator.** It watches for resources and takes action — request certs, renew them 30 days before expiry, recover from failures. You set it up once and forget about it.
- **Let's Encrypt requires domain validation.** Your DNS must point to the Ingress IP *before* requesting a cert, or the HTTP-01 challenge fails.
- **Annotations are how you trigger operator behavior.** `cert-manager.io/cluster-issuer: letsencrypt-prod` is what tells cert-manager to act on this Ingress. This pattern (annotations as instructions) is everywhere in Kubernetes.
- **Helm is how you install operators in real life.** Almost every production add-on (ingress, cert-manager, observability, GitOps tools) ships as a Helm chart. Get comfortable with it now — Lab 5 dives deeper.

---

## 🤔 Reflection Questions

1. Why do you need an Ingress Controller AND an Ingress resource? Why two things?
2. What happens if two Ingress resources define rules for the same hostname?
3. Why does the HTTP-01 challenge require port 80 to be open even if you only want HTTPS?
4. When would you use DNS-01 challenges instead of HTTP-01?
5. If your cert-manager Pod is down for a week and a certificate is set to expire, what happens?

---

## 📚 Further Reading

- [NGINX Ingress Controller Docs](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)
- [AKS Application Gateway Ingress Controller (alternative)](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview)

---

## ➡️ What's Next

In **Lab 5**, we go deep into the tool you've been using to install things: **Helm**. You'll move from *consuming* charts to *building your own* — packaging your application as a reusable, parameterized chart with templates, values, and helpers. By the end, you'll be able to deploy the same app to dev, staging, and prod with `helm upgrade --values prod.yaml`, and you'll understand why Helm is the de facto package manager for Kubernetes.

---

*Author: Prince Chime · Series: Kubernetes on AKS*
