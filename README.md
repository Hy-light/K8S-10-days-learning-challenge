# Kubernetes on AKS — From Basics to Enterprise

> A 10-lab, hands-on journey through Kubernetes on Azure — from your first Pod to a production-grade, GitOps-driven, multi-tenant platform.

[![Made with Markdown](https://img.shields.io/badge/Made_with-Markdown-blue?style=flat-square&logo=markdown)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)
[![AKS](https://img.shields.io/badge/Azure-AKS-0078D4?style=flat-square&logo=microsoftazure&logoColor=white)]()
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29+-326CE5?style=flat-square&logo=kubernetes&logoColor=white)]()

---

## 🎼 What This Series Is

This repository contains 10 progressive, hands-on labs that take you from "what even is a Pod?" to running a multi-tenant, observable, secure, GitOps-driven Kubernetes platform on Azure (AKS).

Each lab is written assuming you're a senior colleague learning alongside me — not a beginner being lectured at. The labs use a consistent **symphony orchestra analogy** to make abstract concepts stick:

- 🎼 Conductor → control plane
- 🎻 Musicians → Pods
- 📋 Sheet music → YAML manifests
- 🎟️ Box office → Services
- 🚪 Smart usher → Ingress Controller
- 📦 Performance package → Helm chart
- 📜 Master library → Git (GitOps)

Once the analogy clicks, the YAML stops feeling like hieroglyphics.

---

## 🎯 Who This Is For

- ✅ Engineers learning Kubernetes from scratch and tired of fragmented tutorials
- ✅ Cloud professionals refreshing knowledge with a hands-on approach
- ✅ Developers transitioning into DevOps or platform engineering
- ✅ Teams looking for a shared reference to onboard new members
- ✅ Anyone preparing for CKA, CKAD, or the AZ-104/AZ-400 with practical context
- ✅ Anyone simply curious

This is **not** a video course or a theoretical reading list. Every lab requires you to run commands, break things, and fix them. That's where the real learning happens.

---

## 📚 The 10-Lab Roadmap

| #   | Lab                                                                                           | Level           | Duration     | What You Build                                                                   |
| --- | --------------------------------------------------------------------------------------------- | --------------- | ------------ | -------------------------------------------------------------------------------- |
| 01  | [AKS Cluster Setup & Your First Pod](lab-01-aks-setup-first-pod.md)                           | 🟢 Beginner     | ~45 min      | A real AKS cluster + your first Pod, deployed imperatively and declaratively     |
| 02  | [Deployments, Services & ReplicaSets](lab-02-deployments-services.md)                         | 🟢 Beginner     | ~60 min      | Self-healing apps, rolling updates, public LoadBalancer Services                 |
| 03  | [ConfigMaps, Secrets & Persistent Storage](lab-03-configmaps-secrets-storage.md)              | 🟢 → 🟡         | ~75 min      | Configuration injection, the Secret base64 truth, Azure Disk persistence         |
| 04  | [Ingress + TLS with cert-manager](lab-04-ingress-tls-cert-manager.md)                         | 🟡 Intermediate | ~90–120 min  | NGINX Ingress + free auto-renewing Let's Encrypt certificates                    |
| 05  | [Helm Charts — Package Your Own App](lab-05-helm-charts.md)                                   | 🟡 Intermediate | ~90–120 min  | Build a chart from scratch, dev/prod values, OCI push to ACR                     |
| 06  | [Kustomize for Multi-Environment Deploys](lab-06-kustomize.md)                                | 🟡 Intermediate | ~75–90 min   | base + overlays/{dev,staging,prod}, strategic merge vs JSON 6902 patches         |
| 07  | [RBAC, Network Policies & Pod Security](lab-07-rbac-network-policies-pod-security.md)         | 🟠 Advanced     | ~90–120 min  | Multi-tenant security: RBAC, default-deny networking, restricted PSS             |
| 08  | [Autoscaling with HPA + KEDA](lab-08-hpa-keda-autoscaling.md)                                 | 🟠 Advanced     | ~90–120 min  | CPU-based HPA, event-driven KEDA, scale-to-zero on Azure Service Bus             |
| 09  | [Observability — Prometheus, Grafana & Loki](lab-09-observability-prometheus-grafana-loki.md) | 🟠 Advanced     | ~120–150 min | Full observability stack, custom dashboards, alerts that fire on real conditions |
| 10  | [GitOps with ArgoCD — Multi-Tenant Platform](lab-10-gitops-argocd-multitenant.md)             | 🔴 Enterprise   | ~150–180 min | ArgoCD, AppProjects, ApplicationSets, App-of-Apps, full disaster recovery        |

**Total time investment:** ~17–20 hours of focused, hands-on work over 10 days (or at your own pace).

---

## 🚀 Quick Start

### 1. Clone the repo

```bash
git clone https://github.com/Hy-light/K8S-10-days-learning-challenge.git
cd K8S-10-days-learning-challenge
```

### 2. Check your prerequisites

You'll need:

- **An Azure account** with an active subscription ([free tier](https://azure.microsoft.com/free) works for most labs)
- **Azure CLI** installed and logged in (`az login`)
- **kubectl** (the lab walks through installing it if you don't have it)
- **Helm 3** (from Lab 4 onwards)
- **Git** + a GitHub/GitLab account (for Lab 10)
- **A domain name** you control (only for Lab 4 if you want real TLS; optional otherwise)

### 3. Start with Lab 1

Open [`lab-01-aks-setup-first-pod.md`](lab-01-aks-setup-first-pod.md) and follow along. Each lab builds on the previous one but is self-contained enough to skip ahead if you already know the basics.

---

## 💸 Cost Awareness

These labs run real Azure infrastructure. With reasonable cleanup, expect:

| Scenario                                                   | Approximate Cost |
| ---------------------------------------------------------- | ---------------- |
| Running one lab and cleaning up                            | < $1             |
| Working through the whole series, cleaning up between labs | $10–25 total     |
| Leaving a cluster running 24/7 for a week                  | $50–100+         |

**Every lab includes a cleanup section.** Run it. Seriously. The most expensive Kubernetes mistake is the cluster you forgot you left running.

For the cheapest path:

- Use `Standard_B2s` nodes (2 vCPU, 4 GB RAM) for Labs 1–6
- Scale up only when needed (Labs 7–10)
- Delete the entire resource group when you take breaks: `az group delete --name <rg-name> --yes`

---

## 🧠 How Each Lab Is Structured

Every lab follows the same shape so you know exactly what to expect:

1. **🎼 ELI5** — The orchestra analogy for the concept you're about to learn
2. **🎯 Learning Objectives** — What you'll be able to do by the end
3. **📋 Prerequisites** — Tools, accounts, prior labs
4. **🧠 Key Concepts** — The 3–5 ideas you need before touching kubectl
5. **🚀 Step-by-Step Instructions** — Numbered steps with full YAML inline
6. **✅ Validation Checklist** — How to confirm everything worked
7. **🧹 Cleanup** — Tear down to avoid charges
8. **🎓 What You Actually Learned** — Synthesis of the deeper lessons
9. **🤔 Reflection Questions** — To test your understanding
10. **📚 Further Reading** — Official docs and deeper rabbit holes
11. **➡️ What's Next** — Teaser for the following lab

---

## 🛠️ Tech Stack Covered

By the end of this series, you'll have hands-on experience with:

**Core Kubernetes:** Pods, Deployments, ReplicaSets, Services, Namespaces, ConfigMaps, Secrets, PersistentVolumes, RBAC, NetworkPolicies, Pod Security Standards

**AKS-specific:** Cluster creation via az CLI, Azure CNI Overlay with Cilium, Azure Disk + Azure Files CSI drivers, Azure Load Balancer, Azure AD (Entra ID) integration, Workload Identity, Azure Container Registry, Cluster Autoscaler

**Ecosystem tools:** Helm, Kustomize, NGINX Ingress Controller, cert-manager, Let's Encrypt, KEDA, Azure Service Bus, kube-prometheus-stack (Prometheus, Grafana, Alertmanager), Loki + Promtail, ArgoCD, ApplicationSets

**GitOps patterns:** App-of-Apps, multi-tenant AppProjects, sync waves, PreSync hooks, self-heal, prune, disaster recovery from Git

---

## 🤝 Contributing

Found a typo? A command that's drifted with a newer Azure CLI version? A clearer way to explain something?

PRs welcome. The best contributions are:

- ✅ Corrections to commands that no longer work
- ✅ Clarifications where a step feels confusing
- ✅ Additional cleanup commands or cost-saving tips
- ✅ Alternative approaches as appendix material
- ❌ Wholesale rewrites or removing the orchestra analogy (it's the spine of the series)

Open an issue first if you're not sure whether something will be accepted.

---

## 📜 License

MIT License — use this material however helps you learn, teach, or grow. Attribution appreciated but not required.

---

## 👋 About the Author

Hi, I'm **Prince Chime** — a Cloud & DevOps Engineer based in Lisbon, Portugal. I've spent the last several years building platforms on Azure and GCP, and I write/record about cloud engineering for engineers who want depth without drowning.

Find me elsewhere:

- 🌐 LinkedIn: [linkedin.com/in/puchime](https://www.linkedin.com/in/puchime/)
- 📺 YouTube: [@princeugochime](https://youtube.com/@princeugochime)
- 🐙 GitHub: [github.com/Hy-light](https://github.com/Hy-light)

If this series helped you, the kindest thing you can do is share it with someone else who's trying to learn Kubernetes. Or drop me a message — I read every one.

---

## 🎼 A Note on the Analogy

You'll notice every lab returns to the symphony orchestra metaphor. That's deliberate.

Kubernetes is a system of well-organized parts that have to work together with timing, coordination, and trust. An orchestra is exactly the same thing in a different medium. Once your brain hooks the concepts onto something concrete — the conductor, the musicians, the box office — the YAML becomes much easier to read, and the architectural decisions much easier to reason about.

The labs will work without the analogy. They work better with it.

🎼 _And the orchestra plays on._

---

## ⭐ If This Helped You

The most useful thing you can do is **star the repo** so others can find it, and **share it** with someone who's trying to learn Kubernetes the hard way. Both take 5 seconds and genuinely help.

Now — go run Lab 1. 👇

**[➡️ Start with Lab 1: AKS Cluster Setup & Your First Pod](lab-01-aks-setup-first-pod.md)**
