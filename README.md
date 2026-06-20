<div align="center">

# 🚢 kubernetes-pods-to-production

### A 6-Phase Hands-On Kubernetes Learning Series

*From writing your first Pod to running a production-grade cluster on AWS —*
*the way it actually works in the real world.*

[![Phases](https://img.shields.io/badge/Phases-6-blue)]()
[![Level](https://img.shields.io/badge/Level-Zero%20to%20Production-green)]()
[![Plaform](https://img.shields.io/badge/Platform-EKS%20-orange)]()
[![Author](https://img.shields.io/badge/Author-Mustafa%20Kanaan-purple)]()

</div>

---

## 👋 A Note From Me

I'm **Mustafa Kanaan**, and I built this repository for one reason: **I couldn't find a Kubernetes resource that actually taught production thinking from day one.**

Most courses teach you what a Pod is, show you `kubectl apply`, and call it "production-ready." They're not. The gap between "I completed a Kubernetes course" and "I can operate Kubernetes in production" is enormous — and most learning material doesn't bridge it.

This series bridges it.

Every concept here is explained the way a senior SRE would explain it to a junior engineer on their first week. Not dumbed down. Not oversimplified. Just clear, honest, and deeply practical — with the exact commands, the exact failure modes, and the exact mental models that matter when things break at 3 AM.

I spent a significant amount of time structuring, writing, and validating every phase. I'm sharing it publicly because **this is the guide I wish existed when I started**, and I genuinely believe it will accelerate anyone's Kubernetes journey by months.

If this helps you — even a little — it was worth sharing.


---


## 📚 What's Inside

This repository contains **6 complete phases**, each building on the last. Every phase includes:
- Full conceptual explanation with real-world context
- Production rules and gotchas that only come from experience
- Complete YAML manifests you can use directly
- Hands-on lab with a step-by-step project
- A checklist to confirm you've mastered the material

---

## 🗺️ The 6-Phase Learning Path

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6
  Core     Network   Storage  Schedule  Observe   Production
```

### Phase 1 — Core Concepts
**File:** `phase1-kubernetes-core-concepts.md`

The absolute foundation. Everything else in Kubernetes builds on these objects.

- What Kubernetes is and the problem it solves (and what it doesn't)
- **Pods** — the atomic unit; why you almost never create them directly
- **Deployments** — the right way to run stateless apps; rolling updates; rollbacks
- **Services** — stable networking for ephemeral Pods; ClusterIP, NodePort, LoadBalancer
- **Namespaces** — logical isolation; resource quotas; team boundaries
- **kubectl** — your primary tool; 20+ essential commands with real use cases

---

### Phase 2 — Networking, Config & Secrets
**File:** `phase2-kubernetes-networking-config.md`

How traffic flows in and out of the cluster, and how apps get their configuration.

- **Ingress** — routing external HTTP/HTTPS to internal Services; path and host rules
- **DNS** — how `service.namespace.svc.cluster.local` works; CoreDNS
- **NetworkPolicy** — firewalls between Pods; default-deny patterns
- **ConfigMaps** — externalize configuration; environment variables vs. volume mounts
- **Secrets** — sensitive data handling; base64 encoding; imagePullSecrets
- **Environment variable injection** — the correct patterns and the common mistakes

---

### Phase 3 — Storage & Stateful Workloads
**File:** `phase3-kubernetes-storage-statefulsets.md`

Everything Pods generate disappears when they restart — unless you use persistent storage.

- **Volumes** — ephemeral storage; emptyDir; sharing data between containers
- **PersistentVolumes (PV)** — cluster-level storage resources
- **PersistentVolumeClaims (PVC)** — how Pods request storage without knowing the backend
- **StorageClass** — dynamic provisioning; the AWS EBS, GCP PD, and Azure Disk story
- **StatefulSets** — ordered deployment; stable Pod identities; sticky storage
- **When to use StatefulSets vs. Deployments** — the decision framework

---

### Phase 4 — Scheduling, Autoscaling & Resource Management
**File:** `phase4-kubernetes-scheduling-autoscaling.md`

How Kubernetes decides where Pods run, and how to make your cluster elastic.

- **Resource Requests & Limits** — the most important thing to get right in production
- **Quality of Service classes** — Guaranteed, Burstable, BestEffort; eviction order
- **Node Affinity & Pod Affinity** — control where Pods land with labels and rules
- **Taints & Tolerations** — reserve nodes for specific workloads
- **Horizontal Pod Autoscaler (HPA)** — auto-scale Pods based on CPU, memory, or custom metrics
- **PodDisruptionBudgets (PDB)** — protect availability during node maintenance
- **The Scheduler algorithm** — filtering, scoring, and binding explained

---

### Phase 5 — Observability & Troubleshooting
**File:** `phase5-kubernetes-observability-troubleshooting.md`

The phase that separates juniors from seniors. Production will break — this is how you fix it fast.

- **The 3 Pillars of Observability** — Logs, Metrics, Traces
- **The 7-Command Diagnostic Toolkit** — the exact sequence used in real incidents
- **Liveness, Readiness & Startup Probes** — why every production Pod needs all three
- **The 6 Most Common Production Failures:**
  - `CrashLoopBackOff` — exit codes, `--previous` logs, root cause patterns
  - `ImagePullBackOff` — registry auth, tag mistakes, network issues
  - Pending Pod — resource pressure, taints, affinity mismatches
  - Service not reachable — the selector mismatch (90% of cases)
  - `OOMKilled` — memory limit tuning; node-level vs. container-level OOM
  - Deployment stuck — bad image, quota exceeded, probe failures
- **Prometheus + Grafana** — the production monitoring stack; PromQL; alert rules
- **The Full Diagnostic Loop** — the 7-step sequence you follow for every incident

---

### Phase 6 — Helm, RBAC, EKS & ArgoCD
**File:** `phase6-kubernetes-helm-rbac-eks-argocd.md`

The production-grade stack. This is how serious engineering teams run Kubernetes.

- **Helm** — Chart structure, values files, multi-environment deployment, `helm template`, `--atomic` upgrades, rollbacks
- **RBAC** — ServiceAccounts, Roles, ClusterRoles, RoleBindings; `kubectl auth can-i`; least-privilege patterns; CI/CD service accounts
- **Amazon EKS** — managed control plane; `eksctl`; IAM authentication; AWS Load Balancer Controller; EBS CSI Driver; Cluster Autoscaler
- **ArgoCD & GitOps** — Git as the single source of truth; automated sync; self-healing; ArgoCD + Helm; the full CI/CD pipeline
- **The Complete Production Workflow** — from `git push` to live deployment without a single manual `kubectl apply`

---

## 🛠️ Labs & Projects

Each phase includes a hands-on project:

| Project | Phase | What You Build |
|---|---|---|
| Project 1–5 | Phase 1 | Deploy a multi-tier app with Deployments and Services |
| Project 6–7 | Phase 2 | Configure Ingress routing, NetworkPolicy, and ConfigMaps |
| Project 8 | Phase 3 | Deploy a stateful PostgreSQL cluster with persistent storage |
| Project 9 | Phase 4 | Set up HPA with CPU autoscaling and test under load |
| Project 10 | Phase 5 | Deploy 5 broken apps, diagnose all 5, fix all 5 |
| Project 11 | Phase 6 | Deploy a full stack with Helm + RBAC + ArgoCD |

---

**Prerequisite knowledge:**
- Basic Linux command line
- Basic understanding of containers and Docker
- No prior Kubernetes experience required

---


## 📋 Full Learning Checklist

Use this to track your progress across the entire series:

- [ ] **Phase 1:** Can create Deployments, Services, and use kubectl fluently
- [ ] **Phase 2:** Can configure Ingress, NetworkPolicy, ConfigMaps, and Secrets
- [ ] **Phase 3:** Can provision persistent storage and deploy StatefulSets
- [ ] **Phase 4:** Can configure HPA, resource limits, taints, and affinities
- [ ] **Phase 5:** Can diagnose any of the 6 production failure types using only kubectl
- [ ] **Phase 6:** Can deploy with Helm, enforce RBAC, provision EKS, and set up ArgoCD

**When all boxes are checked, you're production-ready.**

---

## 💡 Key Philosophy of This Series

> *"The difference between a junior and a senior DevOps engineer is not whether they prevent all failures — it's how fast they diagnose and fix them when failures happen."*

> *"No human ever runs `kubectl apply` in production. Git is the only path to production."*

> *"Every production Pod must have both a liveness and a readiness probe. No exceptions."*

> *"Always set resource requests and limits. A Pod without limits is a ticking time bomb."*

These aren't opinions — they're the rules that protect uptime at scale.

---

## 🤝 Contributing

Found a mistake? Have a better explanation for something? Know a real-world scenario that should be added?

Pull requests are welcome. Please keep the spirit of the series: **practical, production-focused, and clear.**


---

<div align="center">

**Built with care by [Mustafa Kanaan](https://github.com/mustafa-kanaan)**

*If this series helped you land a job, pass a certification, or survive a production incident —*
*that's exactly why I wrote it.* ⭐

</div>
