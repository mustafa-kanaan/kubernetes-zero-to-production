# Phase 6 — Helm, RBAC, EKS & ArgoCD (GitOps)

> **Learning Path:** Zero → Production-Ready | **Phase:** 6 of 6 ✅
> **Prerequisite:** Phase 5 — Observability & Troubleshooting complete

---

## Welcome to the Final Phase

You've built everything by hand so far — YAML files for Deployments, Services, ConfigMaps, Secrets, Ingress, HPA, PVCs. You understand exactly how each piece works. Now you learn how professionals manage all of it at scale.

Phase 6 has four parts that come together at the end into a single professional workflow:

```
Part 1: Helm      → Bundle your YAML into a versioned, reusable package
Part 2: RBAC      → Control exactly who can do what in the cluster
Part 3: EKS       → Run Kubernetes on AWS with a managed control plane
Part 4: ArgoCD    → Let Git drive every deployment automatically
```

By the end, you'll have the workflow used by SRE teams at thousands of companies.

---

## Part 1 — Helm: The Package Manager for Kubernetes

### Why Helm Exists

By now you've written dozens of YAML files. Imagine you need to deploy the same application to three environments: dev, staging, and production — each with different image tags, replica counts, resource limits, and hostnames. That's 30+ YAML files that are 90% identical.

When a new version comes out, you update 30 files. When you need to roll back, you search through Git history across 30 files. When a new engineer joins, they spend days understanding what all 30 files do together.

**Helm solves this.** Helm is to Kubernetes what `apt` or `npm` is to software packages — it bundles all your YAML into one versioned, configurable unit called a **Chart**.

```
Analogy:
  Helm   is to Kubernetes   what  apt/yum  is to Linux
  Chart  is to Helm         what  .deb     is to apt
  Release is to Helm        what  installed package is to apt
```

### Core Helm Concepts

| Term | What It Is |
|---|---|
| **Chart** | A directory of YAML templates + default values that describes a complete application |
| **Release** | One deployment of a Chart into a cluster (same Chart can have multiple releases) |
| **Values** | The configuration variables that customize a Chart for a specific environment |
| **Repository** | A hosted collection of Charts (like Docker Hub but for Charts) |
| **Revision** | A numbered snapshot of a Release's history — enables rollback |

### Chart Directory Structure

```
my-app/                    ← Chart root directory
├── Chart.yaml             ← Chart metadata (name, version, description)
├── values.yaml            ← Default values for template variables
├── values-dev.yaml        ← Override values for dev environment
├── values-staging.yaml    ← Override values for staging environment
├── values-prod.yaml       ← Override values for production environment
└── templates/             ← YAML templates with {{ }} variable substitution
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── hpa.yaml
    └── _helpers.tpl       ← Reusable template snippets
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: "E-commerce web application"
type: application
version: 2.1.0         # Chart version — bump this when you change the Chart
appVersion: "1.4.2"    # Application version — the version of the app being packaged
```

**Critical distinction:**
- `version` = the Chart itself (template changes, new parameters, etc.)
- `appVersion` = the app code inside (Docker image version)

### values.yaml — Your Configuration Interface

Instead of hardcoding values in YAML, you extract them as variables:

```yaml
# values.yaml — the defaults
replicaCount: 2

image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: "myapp.example.com"
  tls: false

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### Templates — Using Variables in YAML

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
        {{- if .Values.autoscaling.enabled }}
        # HPA will manage replicas — override the deployment count
        {{- end }}
```

```yaml
# templates/hpa.yaml — conditionally rendered based on values
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Release.Name }}-{{ .Chart.Name }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

The `{{- if .Values.autoscaling.enabled }}` block means the HPA YAML is only generated when `autoscaling.enabled: true` — the file doesn't get applied otherwise.

### Multi-Environment Deployment — The Power of Values Files

```yaml
# values-prod.yaml — only the differences from defaults
replicaCount: 5

image:
  tag: "1.4.2"          # Production runs the release tag

ingress:
  host: "myapp.com"
  tls: true             # Production uses HTTPS

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "1Gi"

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 50
```

```bash
# Deploy to dev (uses defaults from values.yaml)
helm install myapp-dev ./my-app   --namespace dev   --create-namespace

# Deploy to staging (override specific values)
helm install myapp-staging ./my-app   --namespace staging   -f values-staging.yaml

# Deploy to production (full production values)
helm install myapp-prod ./my-app   --namespace production   -f values-prod.yaml

# All three are independent Releases from the same Chart
helm list --all-namespaces
# NAME            NAMESPACE   REVISION  STATUS    CHART       APP VERSION
# myapp-dev       dev         1         deployed  my-app-2.1.0  1.4.2
# myapp-staging   staging     1         deployed  my-app-2.1.0  1.4.2
# myapp-prod      production  1         deployed  my-app-2.1.0  1.4.2
```

### Essential Helm Commands

```bash
# Installation & management
helm install <release-name> <chart> -n <namespace> -f values.yaml
helm upgrade <release-name> <chart> -n <namespace> -f values.yaml
helm uninstall <release-name> -n <namespace>

# Status & history
helm list -n <namespace>                       # All releases in namespace
helm list --all-namespaces                     # All releases cluster-wide
helm status <release-name> -n <namespace>      # Current state
helm history <release-name> -n <namespace>     # Full revision history

# Rollback
helm rollback <release-name> 3 -n <namespace>  # Roll back to revision 3
helm rollback <release-name> -n <namespace>    # Roll back to previous revision

# Debugging — the most important dev command
helm template <release-name> <chart> -f values.yaml  # Show YAML that would be applied
helm lint <chart>                              # Validate chart structure
helm dry-run install <release> <chart>         # Full dry run with cluster context

# Using public charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/postgresql
helm show values bitnami/postgresql | head -50  # See all configurable values
helm install my-db bitnami/postgresql   --set auth.postgresPassword=secret   --set primary.persistence.size=20Gi
```

### `helm template` — Your Best Debugging Friend

Before deploying anything, run `helm template` to see the exact YAML that Helm will generate. If something deploys wrong, this shows you precisely what was applied — with all variables resolved.

```bash
helm template myapp-prod ./my-app -f values-prod.yaml

# Output:
# ---
# Source: my-app/templates/deployment.yaml
# apiVersion: apps/v1
# kind: Deployment
# metadata:
#   name: myapp-prod-my-app
# spec:
#   replicas: 5
#   ...
# ---
# Source: my-app/templates/hpa.yaml
# apiVersion: autoscaling/v2
# kind: HorizontalPodAutoscaler
# ...
```

### `--atomic` Flag — Safe Deployments

```bash
helm upgrade myapp-prod ./my-app   -n production   -f values-prod.yaml   --atomic \        # If upgrade fails health checks, auto-rollback
  --timeout 5m      # Wait up to 5 minutes for Pods to become Ready

# With --atomic:
# If the new version fails readiness probes within 5 minutes,
# Helm automatically rolls back to the previous revision.
# Production stays healthy — no manual intervention needed.
```

---

## Part 2 — RBAC: Role-Based Access Control

### Why RBAC Matters

Without RBAC, anyone with `kubectl` access to your cluster can do anything — delete Pods, read Secrets containing database passwords, modify production Deployments, or drain nodes. In a company with dozens of engineers, this is unacceptable.

RBAC answers the question: **who can do what to which resources?**

```
Real-world scenario:
  - Junior devs: read-only access to their team's namespace
  - Senior devs: full access to dev/staging namespaces
  - CI/CD pipeline: permission to update Deployments only
  - On-call SRE: full access to production (read + write)
  - Security team: read-only access to all namespaces, cluster-wide
```

### The Four RBAC Objects

```
ServiceAccount  →  Who is doing the action?
Role/ClusterRole  →  What can they do?
RoleBinding/ClusterRoleBinding  →  Connect WHO to WHAT
```

| Resource | Scope | Use Case |
|---|---|---|
| `Role` | Single namespace | "Read Pods in the production namespace" |
| `ClusterRole` | Entire cluster | "Read all Pods across all namespaces" |
| `RoleBinding` | Single namespace | Bind a User/SA to a Role within one namespace |
| `ClusterRoleBinding` | Entire cluster | Bind a User/SA to a ClusterRole across everything |

### ServiceAccount — The Identity for Pods

A ServiceAccount is an identity that Pods use to authenticate with the Kubernetes API. When your CI/CD pipeline or an app inside the cluster needs to talk to Kubernetes (create Pods, update Deployments), it authenticates using a ServiceAccount.

```yaml
# Step 1: Create the ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployment-manager
  namespace: production
```

### Role — Define the Permissions

```yaml
# Step 2: Define what this identity is allowed to do
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-updater
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]  # Can read and update Deployments

- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]            # Can only read Pods (no delete, no exec)

- apiGroups: [""]
  resources: ["secrets"]
  verbs: []                                  # No access to Secrets
```

**Available verbs:** `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

### RoleBinding — Connect Identity to Permissions

```yaml
# Step 3: Bind the ServiceAccount to the Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-deployment-updater
  namespace: production
subjects:
- kind: ServiceAccount
  name: deployment-manager
  namespace: production
roleRef:
  kind: Role
  name: deployment-updater
  apiGroup: rbac.authorization.k8s.io
```

Now the `deployment-manager` ServiceAccount can update Deployments and read Pods in the `production` namespace — but cannot touch Secrets, nodes, or anything in other namespaces.

### ClusterRole & ClusterRoleBinding — Cluster-Wide

```yaml
# ClusterRole: Read all Pods across all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-global
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
---
# Bind to a human user (for SRE on-call dashboard)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sre-readonly
subjects:
- kind: User
  name: "sre-dashboard@company.com"    # Authenticated via AWS IAM on EKS
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-reader-global
  apiGroup: rbac.authorization.k8s.io
```

### Built-In ClusterRoles

Kubernetes ships with several pre-built ClusterRoles you can use directly:

| ClusterRole | What It Grants |
|---|---|
| `cluster-admin` | Full superuser access — everything everywhere |
| `admin` | Full access within a namespace (manage Roles, Deployments, Services, etc.) |
| `edit` | Read/write most resources in a namespace (no RBAC management) |
| `view` | Read-only access to most resources in a namespace |

```bash
# Grant a developer full access to the dev namespace
kubectl create rolebinding dev-access   --clusterrole=edit   --user=developer@company.com   --namespace=dev

# Grant a team lead admin access to staging
kubectl create rolebinding staging-admin   --clusterrole=admin   --user=teamlead@company.com   --namespace=staging
```

### Testing RBAC — `auth can-i`

```bash
# Test if the current user can perform an action
kubectl auth can-i get pods -n production
kubectl auth can-i delete deployments -n production

# Test as a specific ServiceAccount
kubectl auth can-i update deployments   --as=system:serviceaccount:production:deployment-manager   -n production
# Output: yes

kubectl auth can-i delete secrets   --as=system:serviceaccount:production:deployment-manager   -n production
# Output: no

# Get a complete list of what a ServiceAccount can do
kubectl auth can-i --list   --as=system:serviceaccount:production:deployment-manager   -n production
```

`kubectl auth can-i` is your RBAC debugger. When a CI/CD pipeline fails with "forbidden" errors, run this command to immediately identify exactly what permission is missing.

### RBAC for a CI/CD Pipeline

A real-world CI/CD pipeline (GitHub Actions, GitLab CI) needs a ServiceAccount with permission to update Deployments:

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deploy-role
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "update", "patch"]
# No access to secrets, no cluster-wide permissions
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deploy-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: cicd-deployer
  namespace: production
roleRef:
  kind: Role
  name: cicd-deploy-role
  apiGroup: rbac.authorization.k8s.io
```

This follows **least-privilege** — the pipeline can only update the specific resources it needs. If the CI/CD credentials are stolen, the attacker cannot read Secrets or access other namespaces.

---

## Part 3 — Amazon EKS: Kubernetes on AWS

### Why EKS — The Control Plane Problem

Running Kubernetes yourself means managing the control plane: etcd (cluster state database), the API server, the scheduler, the controller manager. If etcd corrupts, your entire cluster is gone. If the API server goes down, no deployments work. This is complex, failure-prone work.

**Amazon EKS** gives you a fully managed control plane:
- AWS runs etcd and the API server across 3 Availability Zones
- AWS handles control plane upgrades, patches, and backup
- You only manage the worker nodes (EC2 instances that run your Pods)
- Integrated with IAM (no separate user management), VPC, ALB, and EBS

### EKS Architecture

```
                    AWS Managed (You don't touch this)
┌─────────────────────────────────────────────────────┐
│  EKS Control Plane (3 AZs)                          │
│  ┌──────────────┐  ┌──────────┐  ┌───────────────┐ │
│  │  API Server  │  │   etcd   │  │  Scheduler    │ │
│  └──────────────┘  └──────────┘  └───────────────┘ │
└─────────────────────────────────────────────────────┘
                        │ kubectl
┌─────────────────────────────────────────────────────┐
│  Your Worker Nodes (EC2 — You manage these)          │
│  ┌────────────────┐  ┌────────────────┐             │
│  │  Node Group 1  │  │  Node Group 2  │             │
│  │  (t3.medium)   │  │  (m5.xlarge)   │             │
│  │  Pods here     │  │  Pods here     │             │
│  └────────────────┘  └────────────────┘             │
└─────────────────────────────────────────────────────┘
                    Your VPC
```

### Creating an EKS Cluster with eksctl

`eksctl` is the official CLI for EKS (made by Weaveworks, endorsed by AWS). One command creates the cluster, node groups, VPC, subnets, IAM roles, and security groups:

```bash
# Install eksctl (macOS)
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Install AWS CLI and configure credentials
aws configure
# AWS Access Key ID: <your-key>
# AWS Secret Access Key: <your-secret>
# Default region: us-east-1

# Create a production-ready EKS cluster
eksctl create cluster   --name production-cluster   --region us-east-1   --nodegroup-name standard-workers   --node-type t3.medium   --nodes 3   --nodes-min 2   --nodes-max 10   --managed                    # Managed Node Group — AWS handles node OS patching

# This takes 15-20 minutes. eksctl will:
# 1. Create a VPC with public/private subnets across 3 AZs
# 2. Create the EKS control plane
# 3. Create a Managed Node Group with 3 t3.medium EC2 instances
# 4. Configure IAM roles for nodes
# 5. Update your ~/.kube/config automatically

# Verify connection
kubectl get nodes
# NAME                          STATUS   ROLES    AGE   VERSION
# ip-192-168-1-10.ec2.internal  Ready    <none>   5m    v1.29.0
# ip-192-168-1-11.ec2.internal  Ready    <none>   5m    v1.29.0
# ip-192-168-1-12.ec2.internal  Ready    <none>   5m    v1.29.0
```

### EKS + IAM = No More Kubernetes User Management

On a local cluster, you manage users with certificates. On EKS, authentication is handled by AWS IAM. Every AWS IAM user/role that you want to give Kubernetes access must be mapped in the `aws-auth` ConfigMap:

```bash
# Grant an IAM user kubectl access
eksctl create iamidentitymapping   --cluster production-cluster   --region us-east-1   --arn arn:aws:iam::123456789012:user/developer-alice   --group system:masters \   # Maps to cluster-admin Kubernetes group
  --username alice

# Verify the mapping was added
kubectl describe configmap aws-auth -n kube-system
```

Then combine with RBAC (Part 2) to give Alice the right Kubernetes permissions.

### EKS Add-Ons — Production Essentials

After creating the cluster, install these critical add-ons:

#### 1. AWS Load Balancer Controller

Required for Ingress to work on EKS. Without it, your Ingress objects do nothing:

```bash
# Add the EKS Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=production-cluster   --set serviceAccount.create=true   --set region=us-east-1   --set vpcId=<your-vpc-id>
```

Now when you create an Ingress, EKS automatically provisions an AWS Application Load Balancer (ALB) in your VPC.

#### 2. EBS CSI Driver

Required for PersistentVolumes backed by AWS EBS (from Phase 3). Without it, `kubectl apply` of PVCs with `storageClassName: gp2` hangs forever:

```bash
# Enable the EBS CSI add-on via eksctl
eksctl create addon   --name aws-ebs-csi-driver   --cluster production-cluster   --region us-east-1

# Verify StorageClass is available
kubectl get storageclass
# NAME            PROVISIONER             RECLAIMPOLICY
# gp2 (default)  kubernetes.io/aws-ebs   Delete
# gp3             ebs.csi.aws.com         Retain
```

#### 3. Cluster Autoscaler

Automatically adds/removes EC2 nodes based on pending Pods:

```bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler   -n kube-system   --set autoDiscovery.clusterName=production-cluster   --set awsRegion=us-east-1
```

When all nodes are full and a new Pod is Pending, the Cluster Autoscaler adds a new EC2 node. When nodes are underutilized for 10+ minutes, it drains and terminates them. Combined with the HPA (Phase 4), this gives you full auto-scaling from Pod level to infrastructure level.

#### 4. Prometheus + Grafana (Phase 5 Revisited)

On EKS, the Prometheus stack works identically to local clusters — same `kube-prometheus-stack` Helm chart:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack   -n monitoring   --create-namespace   --set grafana.adminPassword=YourSecurePassword   --set prometheus.prometheusSpec.retention=30d   --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=gp3   --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

This installs Prometheus with 50GB of EBS-backed persistent storage — your metrics survive Pod restarts and node replacements.

---

## Part 4 — ArgoCD: GitOps for Kubernetes

### What is GitOps?

GitOps is a deployment methodology where **Git is the single source of truth** for everything in the cluster. The desired state of your cluster lives in a Git repository. Any change to the cluster happens through a Git commit — not through running `kubectl apply` manually.

```
Traditional workflow:
  Engineer runs: kubectl apply -f deployment.yaml
  → Direct cluster mutation
  → No audit trail of who changed what
  → No easy rollback to "what was deployed last Tuesday"
  → "Works on my machine" (different engineers have different local files)

GitOps workflow:
  Engineer commits: git push → main branch
  → ArgoCD detects the change
  → ArgoCD applies it to the cluster automatically
  → Full Git history = full deployment history
  → Rollback = git revert
  → Everyone works from the same Git repo
```

### ArgoCD Architecture

```
Git Repository (GitHub/GitLab/Bitbucket)
       │
       │  ArgoCD polls every 3 minutes (or webhook)
       ↓
  ArgoCD (runs inside your Kubernetes cluster)
       │  Compares: Git state vs. Cluster state
       │  If different → applies the difference
       ↓
  Kubernetes Cluster
```

ArgoCD runs as a set of Pods **inside** the cluster it manages. It continuously compares what's in Git with what's actually running. When it detects drift (someone ran `kubectl apply` manually, or a new commit landed), it re-syncs.

### Installing ArgoCD

```bash
# Create the argocd namespace and install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all ArgoCD components to be Ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Get the initial admin password (auto-generated)
kubectl get secret argocd-initial-admin-secret   -n argocd   -o jsonpath="{.data.password}" | base64 -d

# Access the ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080
# Login: admin / <password from above>
```

### Connecting a Git Repository

In ArgoCD UI: **Settings → Repositories → Connect Repo**

Or via CLI:
```bash
# Install ArgoCD CLI
brew install argocd

# Login
argocd login localhost:8080 --username admin --password <password>

# Connect a GitHub repo (HTTPS, public)
argocd repo add https://github.com/yourorg/k8s-configs   --username yourname   --password <github-token>
```

### Creating an Application

An ArgoCD **Application** connects a Git path to a cluster namespace. When Git changes, ArgoCD syncs the cluster.

```yaml
# argocd-application.yaml — deploy the production app
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-webapp
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/yourorg/k8s-configs
    targetRevision: main          # Track the main branch
    path: apps/webapp/production  # Path inside the repo

  destination:
    server: https://kubernetes.default.svc   # Current cluster
    namespace: production

  syncPolicy:
    automated:
      prune: true       # Delete resources removed from Git
      selfHeal: true    # Re-sync if someone manually changes the cluster
    syncOptions:
    - CreateNamespace=true
```

```bash
kubectl apply -f argocd-application.yaml

# Check sync status
argocd app get production-webapp
# Name: production-webapp
# Server: https://kubernetes.default.svc
# Namespace: production
# Status: Synced
# Health: Healthy
# Revision: a3f2c91
```

### ArgoCD + Helm = The Production Stack

ArgoCD natively supports Helm charts. Instead of pointing ArgoCD at raw YAML, you point it at a Helm chart and provide values:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-production
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/yourorg/helm-charts
    targetRevision: main
    path: charts/webapp
    helm:
      valueFiles:
      - values-prod.yaml          # Use production-specific values
      parameters:
      - name: image.tag
        value: "1.4.2"            # Override just the image tag

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Now the complete workflow looks like this:

```
Developer pushes code
    ↓
CI Pipeline (GitHub Actions):
  1. docker build -t myapp:$GIT_SHA .
  2. docker push
  3. yq e '.image.tag = "$GIT_SHA"' -i charts/webapp/values-prod.yaml
  4. git commit && git push  ← the key step
    ↓
ArgoCD detects Git change
    ↓
ArgoCD runs: helm upgrade webapp-prod ./charts/webapp -f values-prod.yaml
    ↓
Kubernetes deploys new Pods
    ↓
Old Pods drain gracefully
    ↓
New version live ✅
```

### The Full GitOps Repository Structure

```
k8s-configs/                ← The GitOps repo — everything about your cluster
├── clusters/
│   ├── production/
│   │   └── argocd-apps/
│   │       ├── webapp.yaml         ← ArgoCD Application for webapp
│   │       ├── database.yaml       ← ArgoCD Application for PostgreSQL
│   │       └── monitoring.yaml     ← ArgoCD Application for Prometheus stack
│   └── staging/
│       └── argocd-apps/
│           └── webapp.yaml
├── charts/
│   └── webapp/                     ← Your custom Helm chart
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       └── templates/
└── manifests/
    └── rbac/                       ← RBAC policies (also managed by ArgoCD)
        ├── cicd-serviceaccount.yaml
        └── dev-team-rolebinding.yaml
```

Everything about your cluster — workloads, RBAC, monitoring, Ingress rules — lives in this repo. **No manual `kubectl apply` in production.** The only way to change production is a Git commit.

### ArgoCD Sync Strategies

| Strategy | Behavior | Use Case |
|---|---|---|
| **Manual** | Requires human to click "Sync" | Sensitive production changes requiring review |
| **Automated** | Syncs on every Git push | Normal deployments, staging environments |
| **Automated + selfHeal** | Also re-syncs if cluster drifts from Git | Production systems that must match Git exactly |
| **Automated + prune** | Deletes resources removed from Git | Keeping cluster clean |

---

## Part 5 — Project 11: The Complete Lab

**Goal:** Deploy a full application stack using all four Phase 6 technologies together.
**Platform:** EKS cluster (or KodeKloud for Helm/ArgoCD portions)
**Time estimate:** 90 minutes

### Step 1 — Build the Helm Chart

```bash
# Create chart structure
mkdir -p fullstack-app/templates
cd fullstack-app

cat > Chart.yaml << 'EOF'
apiVersion: v2
name: fullstack-app
description: Complete application with frontend, backend, and database
version: 1.0.0
appVersion: "1.0.0"
EOF

cat > values.yaml << 'EOF'
replicaCount: 2

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: "app.example.com"
  className: "nginx"

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPU: 70

serviceAccount:
  create: true
  name: ""
EOF
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        release: {{ .Release.Name }}
    spec:
      serviceAccountName: {{ if .Values.serviceAccount.name }}{{ .Values.serviceAccount.name }}{{ else }}{{ .Release.Name }}-sa{{ end }}
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.port }}
        livenessProbe:
          httpGet:
            path: /
            port: {{ .Values.service.port }}
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: {{ .Values.service.port }}
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
```

```bash
# Inspect what the chart will produce
helm template my-release . -f values.yaml

# Install to dev
helm install dev-app . --namespace dev --create-namespace

# Override for production
cat > values-prod.yaml << 'EOF'
replicaCount: 4
image:
  tag: "1.25-alpine"
autoscaling:
  enabled: true
  minReplicas: 4
  maxReplicas: 20
  targetCPU: 60
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"
EOF

helm install prod-app . -f values-prod.yaml --namespace production --create-namespace

# Verify both releases
helm list --all-namespaces
kubectl get pods -n dev
kubectl get pods -n production
```

### Step 2 — Apply RBAC

```bash
# Create the dev-team Role and RoleBinding
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-team-role
  namespace: dev
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]        # Can read secrets, not create/delete
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-team-sa
  namespace: dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: dev-team-sa
  namespace: dev
roleRef:
  kind: Role
  name: dev-team-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Test the permissions
kubectl auth can-i get pods   --as=system:serviceaccount:dev:dev-team-sa   -n dev
# yes

kubectl auth can-i delete secrets   --as=system:serviceaccount:dev:dev-team-sa   -n dev
# no

kubectl auth can-i get pods   --as=system:serviceaccount:dev:dev-team-sa   -n production
# no (namespace-scoped Role)
```

### Step 3 — Upgrade and Rollback with Helm

```bash
# Upgrade dev to a new image tag
helm upgrade dev-app .   --namespace dev   --set image.tag="1.25-alpine"   --atomic   --timeout 3m

# Check revision history
helm history dev-app -n dev
# REVISION  STATUS    CHART               DESCRIPTION
# 1         deployed  fullstack-app-1.0.0 Install complete
# 2         deployed  fullstack-app-1.0.0 Upgrade complete

# Simulate a bad upgrade
helm upgrade dev-app .   --namespace dev   --set image.repository="nginx"   --set image.tag="bad-tag-that-does-not-exist"

# Pod will be in ImagePullBackOff
kubectl get pods -n dev

# Rollback to revision 2
helm rollback dev-app 2 -n dev

# Verify it's healthy again
helm history dev-app -n dev
kubectl get pods -n dev
```

### Step 4 — ArgoCD Sync

```bash
# Install ArgoCD on the cluster
kubectl create namespace argocd
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=Ready pods   -l app.kubernetes.io/name=argocd-server   -n argocd --timeout=300s

# Get admin password
kubectl get secret argocd-initial-admin-secret   -n argocd   -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Login via CLI
argocd login localhost:8080   --username admin   --password $(kubectl get secret argocd-initial-admin-secret -n argocd     -o jsonpath="{.data.password}" | base64 -d)   --insecure

# Create an Application pointing to a public Helm chart
# (using the Bitnami Nginx chart for demonstration)
argocd app create demo-nginx   --repo https://charts.bitnami.com/bitnami   --helm-chart nginx   --revision 15.0.0   --dest-server https://kubernetes.default.svc   --dest-namespace demo   --sync-policy automated   --auto-prune   --self-heal

# Watch it sync
argocd app get demo-nginx
argocd app sync demo-nginx

# Make a change — edit a value, watch ArgoCD apply it
argocd app set demo-nginx --helm-set replicaCount=3
argocd app sync demo-nginx

kubectl get pods -n demo  # Should show 3 replicas
```

### Step 5 — Full Audit: The Complete Picture

```bash
# Everything that's running
helm list --all-namespaces
argocd app list
kubectl get pods --all-namespaces
kubectl get ingress --all-namespaces

# RBAC audit
kubectl get roles,rolebindings --all-namespaces
kubectl get clusterroles,clusterrolebindings   | grep -v "system:"    # Filter out built-in Kubernetes roles

# ArgoCD sync status
argocd app list
# NAME         CLUSTER  NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS
# demo-nginx   in-cluster  demo    default  Synced  Healthy  Auto-Prune  <none>
```

---

## Phase 6 Checklist

### Helm
- [ ] Understand Chart, Release, Values, Repository, and Revision
- [ ] Create a Chart from scratch with `Chart.yaml`, `values.yaml`, and templates
- [ ] Use `{{ .Values.xxx }}` and `{{ .Release.Name }}` in templates
- [ ] Use `{{- if .Values.xxx }}` to conditionally include YAML blocks
- [ ] Deploy the same Chart to 3 environments with different values files
- [ ] Run `helm template` to inspect rendered YAML before deploying
- [ ] Use `helm upgrade --atomic` for safe production deployments
- [ ] Roll back a release with `helm rollback`
- [ ] Install a community chart from Bitnami or another repo

### RBAC
- [ ] Understand ServiceAccount, Role, ClusterRole, RoleBinding, ClusterRoleBinding
- [ ] Create a Role with limited verbs and specific resources
- [ ] Create a RoleBinding connecting a ServiceAccount to a Role
- [ ] Test permissions with `kubectl auth can-i`
- [ ] Test permissions AS a ServiceAccount with `--as=system:serviceaccount:...`
- [ ] Create a least-privilege CI/CD ServiceAccount
- [ ] Use built-in ClusterRoles (`view`, `edit`, `admin`) via RoleBinding

### EKS
- [ ] Understand the EKS control plane vs. worker node separation
- [ ] Create an EKS cluster with `eksctl create cluster`
- [ ] Add IAM user access with `eksctl create iamidentitymapping`
- [ ] Install the AWS Load Balancer Controller
- [ ] Install the EBS CSI Driver for PersistentVolumes
- [ ] Install Cluster Autoscaler
- [ ] Install Prometheus + Grafana with persistent EBS storage

### ArgoCD / GitOps
- [ ] Install ArgoCD on a cluster
- [ ] Connect a Git repository to ArgoCD
- [ ] Create an ArgoCD Application via YAML
- [ ] Enable automated sync with `selfHeal: true` and `prune: true`
- [ ] Point ArgoCD at a Helm chart with custom values
- [ ] Trigger a deployment via Git commit (not kubectl)
- [ ] Observe ArgoCD detect and fix cluster drift

---

## The Complete Production Workflow

This is how production deployments work at companies using Kubernetes properly:

```
1. Developer writes code, opens Pull Request
          ↓
2. CI Pipeline (GitHub Actions / GitLab CI):
   - Runs tests
   - Builds Docker image: docker build -t myapp:$GIT_SHA .
   - Pushes image: docker push myapp:$GIT_SHA
   - Updates Helm values: yq e '.image.tag = "$GIT_SHA"' -i charts/myapp/values-prod.yaml
   - Commits + pushes to Git: git push
          ↓
3. PR is reviewed, approved, merged to main
          ↓
4. ArgoCD detects the new commit in main
          ↓
5. ArgoCD runs: helm upgrade myapp ./charts/myapp -f values-prod.yaml --atomic
          ↓
6. Kubernetes applies rolling update
   - New Pods start, pass readiness probes
   - Traffic gradually shifts from old → new Pods
   - Old Pods drain and terminate
          ↓
7. Prometheus monitors CPU/memory/error rates
   AlertManager fires to Slack if error rate spikes
          ↓
8. If something goes wrong:
   - Emergency: argocd app rollback myapp
   - Proper fix: git revert → push → ArgoCD auto-deploys the revert
```

**No human ever runs `kubectl apply` in production. Git is the only path to production.**

---

## Where to Go From Here

You now have the complete Kubernetes foundation. Here's what senior engineers specialize in next:

| Domain | Technologies |
|---|---|
| **Security** | OPA/Gatekeeper, Kyverno (policy enforcement), Falco (runtime threat detection), cert-manager (TLS automation), Vault (secrets management) |
| **Service Mesh** | Istio or Linkerd — mTLS between all services, traffic shaping, A/B testing, circuit breakers, distributed tracing |
| **Multi-cluster** | ArgoCD ApplicationSets, Cluster API, Crossplane, fleet management across regions |
| **Cost optimization** | Kubecost, Spot instances on EKS, right-sizing with VPA, node bin-packing |
| **Developer Platform** | Backstage (internal developer portal), Crossplane (infrastructure as Kubernetes resources), Platform Engineering |

---

*Phase 6 of 6 — Kubernetes Learning Series Complete ✅ | Mustafa Kanaan*
