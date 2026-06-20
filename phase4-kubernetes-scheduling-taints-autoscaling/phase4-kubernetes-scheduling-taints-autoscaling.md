# Phase 4 — Scheduling, Taints & Autoscaling

> **Learning Path:** Zero → Production-Ready | **Phase:** 4 of 6
> **Prerequisite:** Phase 3 — Configuration, Secrets & Persistent Storage complete

---

## Why This Phase Exists

Phases 1–3 taught you how to run workloads, expose them, configure them, and store their data. But you've been letting Kubernetes decide everything about *where* Pods run and *how many* run at any given time.

In production, those two decisions matter enormously:
- A GPU-intensive ML job landing on a cheap general-purpose node wastes GPU capacity and makes the ML job slow
- All 3 replicas of your API landing on the same node means one node failure takes down the entire service
- A traffic spike hitting your app during a sale event causes timeouts if the cluster can't scale fast enough

This phase gives you the tools to control both decisions:
1. **Scheduling controls** — tell Kubernetes exactly where to place Pods, and where not to
2. **Autoscaling** — let Kubernetes scale Pod count automatically based on real traffic

---

## Part 1 — How the Kubernetes Scheduler Works

Before controlling scheduling, you need to understand what the scheduler does by default.

When a Pod is created, the Kubernetes Scheduler evaluates **every node** in the cluster and picks the best one using a two-stage process:

```
Stage 1 — Filtering (hard rules)
├── Does the node have enough CPU and memory?
├── Does the node match any NodeSelector or NodeAffinity?
├── Does the Pod have Tolerations for the node's Taints?
├── Are the PVCs available in this node's AZ?
└── → Discard any node that fails any filter

Stage 2 — Scoring (soft preferences)
├── Spread Pods across nodes for fault tolerance
├── Prefer nodes with more available resources
├── Prefer nodes where the container image already exists
└── → Pick the highest-scoring remaining node
```

Without any scheduling controls, the scheduler spreads Pods across all available nodes. This is fine for simple apps, but in production you need more control.

---

## Part 2 — Taints and Tolerations

### The Concept

A **Taint** is a label placed on a **node** that says "Pods are not allowed here unless they have permission." A **Toleration** is the permission on a **Pod** that says "I'm allowed onto nodes with this specific taint."

Think of it like a bouncer at a club:
- The taint is the bouncer saying "No entry without a wristband"
- The toleration is the wristband

### Why Taints Exist

In a real production cluster you have different node types:

```
┌─────────────────────────────────────────────────────────┐
│  GPU nodes — expensive, reserved for ML workloads       │
│  taint: gpu=true:NoSchedule                             │
├─────────────────────────────────────────────────────────┤
│  Spot/Preemptible nodes — cheap but can die anytime     │
│  taint: spot=true:NoSchedule                            │
├─────────────────────────────────────────────────────────┤
│  High-memory nodes — reserved for databases             │
│  taint: highmem=true:NoSchedule                         │
├─────────────────────────────────────────────────────────┤
│  Regular nodes — for everything else                    │
│  no taints                                              │
└─────────────────────────────────────────────────────────┘
```

Without taints, a lightweight Nginx Pod could land on your expensive GPU node, wasting capacity. Taints prevent that.

### The Three Taint Effects

| Effect | What It Does |
|---|---|
| `NoSchedule` | New Pods without a toleration will **never** be scheduled here |
| `PreferNoSchedule` | Kubernetes will **try to avoid** scheduling here, but will if no other node is available |
| `NoExecute` | New Pods without a toleration won't be scheduled here, **AND** existing Pods without a toleration are **evicted** |

`NoExecute` is the powerful one — it's what Kubernetes uses internally when a node becomes unhealthy. The node automatically gets a `node.kubernetes.io/not-ready:NoExecute` taint, and Pods that have been running there get evicted after a grace period.

### Adding a Taint to a Node

```bash
# Taint a node — only GPU workloads allowed
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule

# View taints on a node
kubectl describe node gpu-node-1 | grep Taints

# Remove a taint (note the trailing minus sign)
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule-
```

### Adding a Toleration to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training-job
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: training
    image: tensorflow/tensorflow:2.13-gpu
    resources:
      limits:
        nvidia.com/gpu: 1
```

### Critical Concept: Tolerations Allow, They Don't Force

> A toleration does **not** guarantee a Pod will land on a tainted node. It just means the Pod is **permitted** to land there.
>
> If you want to **force** a Pod onto a specific node type, you combine tolerations with **Node Affinity**.

### System Taints — Why `kubectl describe node` Always Shows Taints

Kubernetes automatically applies taints to nodes in certain states. You'll see these on any cluster:

| Taint | Applied When |
|---|---|
| `node.kubernetes.io/not-ready` | Node is unhealthy |
| `node.kubernetes.io/unreachable` | Node is unreachable from the control plane |
| `node.kubernetes.io/memory-pressure` | Node is running low on memory |
| `node.kubernetes.io/disk-pressure` | Node is running low on disk |
| `node-role.kubernetes.io/control-plane:NoSchedule` | Control plane nodes — don't run regular workloads here |

The last one is why your workload Pods never land on control plane nodes in a managed cluster (EKS, GKE, AKS). The control plane is tainted, and your Deployments don't have that toleration.

---

## Part 3 — Node Affinity

While taints tell nodes to repel Pods, **Node Affinity** tells Pods to attract toward (or require) specific nodes. It works on **node labels**.

### Why Not Just NodeSelector?

`nodeSelector` is the simple version — it hard-requires a label match:
```yaml
nodeSelector:
  disktype: ssd
```

Node Affinity is more powerful — it supports:
- Hard requirements (`requiredDuringSchedulingIgnoredDuringExecution`) — Pod won't schedule if not met
- Soft preferences (`preferredDuringSchedulingIgnoredDuringExecution`) — Pod prefers this, but will run elsewhere if needed
- Complex expressions (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`)

### Node Affinity Example — GPU Workload

```yaml
spec:
  affinity:
    nodeAffinity:
      # Hard requirement — MUST land on an SSD node
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      # Soft preference — prefer nodes in us-east-1a
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
```

### What "IgnoredDuringExecution" Means

The long names have a specific meaning:
- `DuringScheduling` — applies when the Pod is being scheduled
- `IgnoredDuringExecution` — if the node's labels change **after** the Pod is running, the Pod is **not** evicted

The counterpart (`RequiredDuringExecution`) would evict Pods if the node no longer matches — this feature exists but is rare in practice.

### Node Labels in AWS EKS

EKS automatically labels every node with useful information you can use for Node Affinity:

```bash
# Common EKS node labels
topology.kubernetes.io/zone: us-east-1a        # Availability zone
node.kubernetes.io/instance-type: m5.xlarge    # EC2 instance type
eks.amazonaws.com/nodegroup: production-nodes  # Node group name
kubernetes.io/arch: amd64                      # CPU architecture
```

```bash
# View all labels on a node
kubectl get node <node-name> --show-labels

# Add a custom label to a node
kubectl label node <node-name> disktype=ssd workload=database
```

---

## Part 4 — Pod Anti-Affinity

**Pod Anti-Affinity** solves a different problem: not where Pods *should* go, but where they should *not* be relative to each other.

The scenario: You have 3 replicas of your API. Without any controls, all 3 could land on the same node. If that node dies, all 3 replicas die at once — full outage. With Pod Anti-Affinity, you tell Kubernetes "never put two replicas of this app on the same node."

```yaml
spec:
  affinity:
    podAntiAffinity:
      # Hard requirement — never put 2 instances of this app on the same node
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - my-api
        topologyKey: kubernetes.io/hostname   # "hostname" = per-node spreading

      # Soft preference — try to spread across AZs too
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - my-api
          topologyKey: topology.kubernetes.io/zone
```

The `topologyKey` controls the scope:
- `kubernetes.io/hostname` — spread across nodes
- `topology.kubernetes.io/zone` — spread across AZs
- `topology.kubernetes.io/region` — spread across regions (rare)

> **Production best practice:** Use `preferredDuringScheduling` for anti-affinity, not `requiredDuringScheduling`. If you use hard anti-affinity and you have 3 replicas but only 2 nodes, the third Pod will forever stay `Pending` — the scheduler physically can't satisfy the constraint.

### Pod Affinity vs. Anti-Affinity

| | Pod Affinity | Pod Anti-Affinity |
|---|---|---|
| Purpose | Co-locate related Pods | Spread Pods apart |
| Use case | App + sidecar on same node, cache near app | HA — never all replicas on one node |
| Risk | Creates scheduling bottleneck if overused | Can block scheduling if too strict |

---

## Part 5 — Resource Quotas and LimitRanges

### The Problem Without Limits

In a shared cluster, one rogue Deployment can consume all available CPU and memory, starving other teams' workloads. Kubernetes solves this with two objects:
- **ResourceQuota** — limits total resource consumption per namespace
- **LimitRange** — sets default and maximum resource requests/limits per container

### ResourceQuota

Applied at the **namespace** level — limits the total resources all workloads in that namespace can consume:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute
    requests.cpu: "20"          # Total CPU requests across all Pods in namespace
    requests.memory: 40Gi       # Total memory requests across all Pods
    limits.cpu: "40"            # Total CPU limits
    limits.memory: 80Gi
    # Object counts
    pods: "50"                  # Max 50 Pods in this namespace
    services: "20"
    persistentvolumeclaims: "10"
    secrets: "30"
    configmaps: "20"
```

```bash
# Check current quota usage
kubectl describe resourcequota production-quota -n production
# Shows: used vs hard limit for each resource
```

### LimitRange

Applied at the **namespace** level — sets defaults and maximum per container. This is critical because if a container in a namespace with a ResourceQuota doesn't specify resource requests/limits, the Pod will be **rejected**:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:                    # Applied if container doesn't specify limits
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:             # Applied if container doesn't specify requests
      cpu: "100m"
      memory: "128Mi"
    max:                        # Container can never exceed these
      cpu: "4"
      memory: "8Gi"
    min:                        # Container must request at least this
      cpu: "50m"
      memory: "64Mi"
```

### ResourceQuota vs. LimitRange

| | ResourceQuota | LimitRange |
|---|---|---|
| Scope | Namespace total | Per container |
| Purpose | Prevent a namespace consuming too much | Set defaults and caps per container |
| Common use | Multi-tenant clusters | Ensure no container goes unbounded |
| Applied when | Pod is scheduled | Pod spec is admitted |

---

## Part 6 — Horizontal Pod Autoscaler (HPA)

### The Core Concept

The **HPA** automatically adjusts the replica count of a Deployment (or StatefulSet) based on observed metrics. The most common metric is CPU utilization, but it can also scale on memory, custom metrics (requests-per-second, queue length), or external metrics.

```
Traffic spikes → CPU goes to 90% → HPA adds Pods → CPU drops to ~50%
Traffic drops  → CPU drops to 20% → HPA removes Pods → saves money
```

### How the HPA Control Loop Works

The HPA runs a control loop every **15 seconds** (configurable). It:
1. Fetches current CPU/memory metrics from the Metrics Server
2. Calculates the desired replica count:
   \[
   	ext{desiredReplicas} = \lceil 	ext{currentReplicas} 	imes rac{	ext{currentMetricValue}}{	ext{targetMetricValue}} ceil
   \]
3. Applies the new replica count if it changed

Example:
- `currentReplicas = 2`
- `currentCPU = 90%`
- `targetCPU = 50%`
- `desiredReplicas = ceil(2 × 90/50) = ceil(3.6) = 4`

The HPA would scale from 2 → 4 replicas.

### Prerequisites — Metrics Server Must Be Running

The HPA queries the **Metrics Server** for CPU and memory data. On EKS, install it:

```bash
# Install Metrics Server on EKS
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify it's running
kubectl get pods -n kube-system | grep metrics-server

# Test it works — should show CPU/memory for all nodes
kubectl top nodes

# Test it works — should show CPU/memory for all pods
kubectl top pods -n production
```

On KodeKloud playgrounds, Metrics Server is usually pre-installed. If `kubectl top` fails, the HPA won't work.

### Creating an HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2              # Never scale below 2 (high availability)
  maxReplicas: 10             # Never scale above 10 (cost cap)
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50    # Scale when average CPU exceeds 50%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70    # Also scale when memory exceeds 70%
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30    # Wait 30s before scaling up
      policies:
      - type: Pods
        value: 4                         # Add at most 4 Pods per scale event
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 minutes before scaling down
      policies:
      - type: Percent
        value: 25                        # Remove at most 25% of Pods per scale event
        periodSeconds: 60
```

Or imperatively:

```bash
kubectl autoscale deployment webapp   --min=2 --max=10 --cpu-percent=50   -n production
```

### Scale-Down Stabilization — Why It Matters

The `stabilizationWindowSeconds: 300` for scale-down is critical. Without it:
- Traffic drops for 15 seconds → HPA removes Pods immediately
- Traffic spikes again → HPA scrambles to add Pods back
- Users see errors during the 30–60 seconds it takes new Pods to start

With a 5-minute stabilization window, the HPA only scales down if the metric stays low for the full 5 minutes — preventing thrashing.

### Watching the HPA in Action

```bash
# Watch HPA status — updates every 15 seconds
kubectl get hpa -n production -w

# Detailed HPA status
kubectl describe hpa webapp-hpa -n production
# Shows: current replicas, desired replicas, last scale time, current metrics
```

### Resource Requests Are Required for HPA

> If your Pods don't have `resources.requests.cpu` set, the HPA **cannot calculate utilization percentage** and will show:
> ```
> unknown/50%
> ```
> The HPA will not scale. Always set resource requests on every container.

---

## Part 7 — Load Testing with k6

You can define an HPA and set targets all day, but you won't know if it actually works until you hit the app with real traffic. **k6** is the standard tool for generating load in Kubernetes environments.

### Why k6?

k6 is a developer-friendly load testing tool where test scripts are written in JavaScript. It integrates cleanly with Kubernetes because you can run k6 as a Pod inside the cluster — meaning the load hits your Service's ClusterIP directly, not through an Ingress or load balancer.

### Installing k6

```bash
# macOS
brew install k6

# Linux
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg   --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main"   | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6
```

### Writing a Load Test Script

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 10 },    // Ramp up to 10 VUs over 1 minute
    { duration: '3m', target: 50 },    // Ramp up to 50 VUs and hold for 3 minutes
    { duration: '2m', target: 100 },   // Spike to 100 VUs
    { duration: '2m', target: 50 },    // Scale back down to 50
    { duration: '1m', target: 0 },     // Ramp down to 0
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests must complete under 500ms
    http_req_failed: ['rate<0.01'],    // Error rate must stay below 1%
  },
};

export default function () {
  const res = http.get('http://webapp-service.production.svc.cluster.local/');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);  // Each VU waits 1 second between requests
}
```

### Running the Load Test

```bash
# Run the test
k6 run load-test.js

# Run with real-time output summary
k6 run --out json=results.json load-test.js
```

### What to Watch During a Load Test

Run these in separate terminals while the load test is running:

```bash
# Terminal 1 — Watch HPA scaling decisions
kubectl get hpa webapp-hpa -n production -w

# Terminal 2 — Watch Pod count change in real time
kubectl get pods -n production -w

# Terminal 3 — Watch resource consumption
watch kubectl top pods -n production
```

### Interpreting HPA Behavior During Load

```
# What you should see during a successful test:

# At test start (low load)
NAME         REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
webapp-hpa   Deployment/webapp     8%/50%    2         10        2

# After ramp-up (50 VUs)
webapp-hpa   Deployment/webapp     72%/50%   2         10        2   ← HPA sees 72% > 50%
webapp-hpa   Deployment/webapp     72%/50%   2         10        3   ← Scales to 3
webapp-hpa   Deployment/webapp     55%/50%   2         10        3
webapp-hpa   Deployment/webapp     48%/50%   2         10        4   ← Scales to 4

# After spike (100 VUs)
webapp-hpa   Deployment/webapp     85%/50%   2         10        7   ← Scales to 7

# After ramp-down
webapp-hpa   Deployment/webapp     40%/50%   2         10        7   ← Stays at 7 (stabilization)
# ... 5 minutes later ...
webapp-hpa   Deployment/webapp     20%/50%   2         10        4   ← Scales down
webapp-hpa   Deployment/webapp     12%/50%   2         10        2   ← Back to minimum
```

---

## Part 8 — Production Issues

### ⚠️ Issue 1: Pod Stuck in Pending — Taint Not Tolerated

**Scenario:** You deploy a Pod and it stays `Pending`. `kubectl describe pod` shows:

```
0/3 nodes are available: 1 node has taint {gpu: true} that the pod didn't tolerate,
2 nodes had taints {node-role.kubernetes.io/control-plane} that the pod didn't tolerate.
```

**What it means:** The scheduler checked all nodes. The GPU node has a taint the Pod doesn't tolerate. The other 2 nodes are control plane nodes (always tainted). There are no regular worker nodes available in the cluster.

**Diagnosis and fix:**

```bash
# Step 1: Check all node taints
kubectl describe nodes | grep -A3 Taints

# Step 2: Check how many worker nodes exist
kubectl get nodes

# In KodeKloud labs: often only 1 control-plane + 1 worker
# If both are tainted, you have nowhere to schedule

# Fix option A: Add a toleration to the Pod spec
# Fix option B: Remove the taint from the node (if it's a test environment)
kubectl taint nodes <node-name> gpu=true:NoSchedule-

# Fix option C: On a single-node test cluster, remove the control-plane taint
# (NOT for production — this is only for local testing)
kubectl taint nodes <node-name> node-role.kubernetes.io/control-plane:NoSchedule-
```

---

### ⚠️ Issue 2: HPA Showing `<unknown>/50%`

**Scenario:** You create an HPA but it shows `<unknown>/50%` and never scales.

**Root causes:**
1. Metrics Server is not installed
2. Pods don't have `resources.requests.cpu` set
3. Metrics Server is installed but not yet ready (takes 1–2 minutes after install)

**Diagnosis:**

```bash
# Check Metrics Server
kubectl get pods -n kube-system | grep metrics-server

# If metrics-server is running, can you get metrics?
kubectl top pods -n production
# If this fails: Metrics Server is broken or not ready

# Check if Pods have resource requests set
kubectl get deployment webapp -n production -o yaml | grep -A4 resources
# Should show:
#   resources:
#     requests:
#       cpu: "100m"

# Check HPA events for the exact error
kubectl describe hpa webapp-hpa -n production
# Scroll to Events at the bottom
```

**Fix:**

```bash
# Add resource requests to the Deployment
kubectl patch deployment webapp -n production --patch '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "app",
          "resources": {
            "requests": {"cpu": "100m", "memory": "128Mi"},
            "limits": {"cpu": "500m", "memory": "256Mi"}
          }
        }]
      }
    }
  }
}'

# Wait for new Pods to roll out, then check HPA
kubectl get hpa -n production
```

---

### ⚠️ Issue 3: HPA Not Scaling Despite High CPU

**Scenario:** CPU shows `85%/50%` but the HPA doesn't add Pods.

**Root causes:**
1. `maxReplicas` is already reached — the HPA can't go higher
2. A stabilization window is preventing scale-up
3. The HPA is in a cooldown period from a recent scale event

**Diagnosis:**

```bash
kubectl describe hpa webapp-hpa -n production

# Look at the Events section:
# "New size: 10; reason: cpu resource utilization (percentage of request)
#  above target" — means it's AT maxReplicas already
# "DesiredReplicas" > maxReplicas? You need to raise maxReplicas
```

**Fix:**

```bash
# Raise the max replicas limit
kubectl patch hpa webapp-hpa -n production   --patch '{"spec": {"maxReplicas": 20}}'
```

---

### ⚠️ Issue 4: All Pods on Same Node — No Anti-Affinity

**Scenario:** A node dies and your entire service goes down. `kubectl get pods` shows all replicas were on the same node.

```bash
# Verify — check which node each Pod ran on
kubectl get pods -n production -o wide
# If all pods show the same NODE column: this is the problem
```

**Fix — Add Pod Anti-Affinity to the Deployment:**

```bash
kubectl patch deployment webapp -n production --patch '
{
  "spec": {
    "template": {
      "spec": {
        "affinity": {
          "podAntiAffinity": {
            "preferredDuringSchedulingIgnoredDuringExecution": [{
              "weight": 100,
              "podAffinityTerm": {
                "labelSelector": {
                  "matchExpressions": [{
                    "key": "app",
                    "operator": "In",
                    "values": ["webapp"]
                  }]
                },
                "topologyKey": "kubernetes.io/hostname"
              }
            }]
          }
        }
      }
    }
  }
}'

# Force a rollout to re-spread the Pods
kubectl rollout restart deployment/webapp -n production

# Verify Pods are now on different nodes
kubectl get pods -n production -o wide
```

---

## Part 9 — Hands-On Lab: Projects 8 & 9

**Platform:** KodeKloud Kubernetes playground
**Time estimate:** 60 minutes
**Goal:** Control scheduling, prove anti-affinity works, and watch HPA respond to real k6 load

### Project 8 — Taints, Tolerations & Node Affinity (30 min)

#### Step 1 — Inspect Your Nodes

```bash
# See all nodes, their roles, and their labels
kubectl get nodes --show-labels

# Check taints on each node
kubectl describe nodes | grep -E "Name:|Taints:"

# Note the node names — you'll use them throughout this lab
```

#### Step 2 — Taint a Node and Observe Scheduling Failure

```bash
# Pick a worker node and taint it
kubectl taint nodes <worker-node-name> dedicated=gpu:NoSchedule

# Try to deploy a Pod — it will get stuck in Pending
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: regular-pod
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
EOF

# Watch it stay Pending
kubectl get pod regular-pod -w

# Read why it can't schedule
kubectl describe pod regular-pod | grep -A5 Events
# You'll see: 1 node had taint {dedicated: gpu} that the pod didn't tolerate
```

#### Step 3 — Add a Toleration and Observe Scheduling Success

```bash
# Delete the stuck pod
kubectl delete pod regular-pod

# Create a pod WITH the toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tolerated-pod
  namespace: default
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
EOF

# This one should schedule and run
kubectl get pod tolerated-pod -w
```

#### Step 4 — Label a Node and Use Node Affinity

```bash
# Add a custom label to a worker node
kubectl label node <worker-node-name> disktype=ssd environment=production

# Deploy a Pod that REQUIRES this label
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  namespace: default
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  tolerations:                   # Also needs the toleration for the taint we added
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
EOF

# Verify it landed on the labeled node
kubectl get pod ssd-pod -o wide
# NODE column should show <worker-node-name>
```

#### Step 5 — Test Pod Anti-Affinity (requires at least 2 nodes)

```bash
# Remove the taint first so we can schedule normally
kubectl taint nodes <worker-node-name> dedicated=gpu:NoSchedule-

# Deploy 3 replicas WITH pod anti-affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-webapp
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ha-webapp
  template:
    metadata:
      labels:
        app: ha-webapp
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: ha-webapp
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
EOF

# Verify Pods are spread across different nodes
kubectl get pods -o wide -l app=ha-webapp
# NODE column should show DIFFERENT nodes for each Pod
```

#### Step 6 — Clean Up Project 8

```bash
kubectl delete deployment ha-webapp
kubectl delete pod ssd-pod tolerated-pod
kubectl label node <worker-node-name> disktype- environment-
```

---

### Project 9 — HPA + k6 Load Testing (30 min)

#### Step 1 — Deploy the Target App with Resource Requests

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loadtest-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: loadtest-app
  template:
    metadata:
      labels:
        app: loadtest-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "100m"       # REQUIRED for HPA to work
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: loadtest-service
  namespace: default
spec:
  selector:
    app: loadtest-app
  ports:
  - port: 80
    targetPort: 80
EOF
```

#### Step 2 — Verify Metrics Server is Working

```bash
# Should show CPU/memory for pods
kubectl top pods

# If this fails, install Metrics Server first:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# Wait 2 minutes then try again
```

#### Step 3 — Create the HPA

```bash
kubectl autoscale deployment loadtest-app   --min=2   --max=10   --cpu-percent=50

# Verify the HPA was created and is reading metrics
kubectl get hpa loadtest-app -w
# Wait until TARGETS shows a real percentage (not <unknown>)
# This can take up to 90 seconds
```

#### Step 4 — Run the Load Test

Save this as `load-test.js`:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },
    { duration: '1m',  target: 50 },
    { duration: '30s', target: 0 },
  ],
};

export default function () {
  const res = http.get('http://loadtest-service.default.svc.cluster.local/');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(0.5);
}
```

```bash
# Run the test (in a separate terminal)
k6 run load-test.js
```

#### Step 5 — Watch the HPA Scale in Real Time

```bash
# Terminal 1 — HPA decisions
kubectl get hpa loadtest-app -w

# Terminal 2 — Pod count
kubectl get pods -l app=loadtest-app -w

# Terminal 3 — Resource consumption
watch kubectl top pods -l app=loadtest-app
```

#### Step 6 — Clean Up Project 9

```bash
kubectl delete deployment loadtest-app
kubectl delete service loadtest-service
kubectl delete hpa loadtest-app
```

---

## Essential kubectl for This Phase

```bash
# Nodes and Taints
kubectl describe nodes | grep -E "Name:|Taints:"     # All node taints at once
kubectl taint nodes <node> key=value:effect           # Add taint
kubectl taint nodes <node> key=value:effect-          # Remove taint (trailing dash)
kubectl label node <node> key=value                   # Add node label
kubectl label node <node> key-                        # Remove node label
kubectl get nodes --show-labels                       # See all node labels

# HPA
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=50
kubectl get hpa -n <namespace>                        # List HPAs
kubectl describe hpa <name> -n <namespace>            # See current metrics and events
kubectl get hpa <name> -n <namespace> -w              # Watch scaling decisions

# Metrics
kubectl top nodes                                     # CPU/memory per node
kubectl top pods -n <namespace>                       # CPU/memory per pod
kubectl top pods --sort-by=cpu -n <namespace>         # Sort by CPU usage

# Scheduling troubleshooting
kubectl get pods -o wide                              # Shows which node each Pod is on
kubectl describe pod <name> | grep -A10 Events        # Why is Pod Pending?
```

---

## Phase 4 Checklist

- [ ] Understand the two-stage Kubernetes scheduler (Filter → Score)
- [ ] Understand the 3 Taint effects: NoSchedule, PreferNoSchedule, NoExecute
- [ ] Add a taint to a node and observe a Pod fail to schedule
- [ ] Add a toleration to a Pod and observe it schedule on a tainted node
- [ ] Understand system taints (control-plane, not-ready, memory-pressure)
- [ ] Understand the difference between Node Affinity and nodeSelector
- [ ] Create a hard Node Affinity requirement using `requiredDuringScheduling`
- [ ] Create a soft Node Affinity preference using `preferredDuringScheduling`
- [ ] Understand `topologyKey` in Pod Anti-Affinity
- [ ] Verify Pod Anti-Affinity spreads Pods across different nodes
- [ ] Understand why tolerations allow but don't force — combine with Node Affinity to force
- [ ] Create a ResourceQuota on a namespace
- [ ] Create a LimitRange with defaults and maximums
- [ ] Install Metrics Server and verify `kubectl top` works
- [ ] Create an HPA with min/max replicas and CPU target
- [ ] Confirm HPA shows a real percentage (not `<unknown>`)
- [ ] Run a k6 load test and watch the HPA scale in real time
- [ ] Confirm scale-down stabilization prevents thrashing

---

## What's Next — Phase 5 Preview

Phase 5 covers **Observability & Troubleshooting** — the skills you use every single day as a DevOps engineer:

- **The 3 Pillars of Observability** — Logs, Metrics, Traces
- **kubectl log inspection** — multi-container Pods, previous container crashes, following logs
- **Prometheus + Grafana on Kubernetes** — deploying the full monitoring stack
- **Production incident methodology** — the exact diagnostic flow for the 7 most common production failures
- **kubectl exec for live debugging** — getting inside running containers to diagnose problems
- **CrashLoopBackOff, OOMKilled, ImagePullBackOff** — diagnosing and fixing the most common Pod failure modes

---

*Phase 4 of 6 — Kubernetes Learning Series | Mustafa Kanaan*
