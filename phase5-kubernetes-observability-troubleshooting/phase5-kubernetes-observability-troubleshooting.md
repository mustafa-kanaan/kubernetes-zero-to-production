# Phase 5 — Observability & Troubleshooting

> **Learning Path:** Zero → Production-Ready | **Phase:** 5 of 6
> **Prerequisite:** Phase 4 — Scheduling, Taints & Autoscaling complete

---

## Why This Phase Exists

Everything built in Phases 1–4 is designed to work. But in production, things will break.

A Pod crashes at 3 AM. An app becomes unreachable right before a flash sale. A deployment hangs halfway through a rollout. A node runs out of memory and starts evicting Pods. The database stops accepting connections.

The difference between a junior and a senior DevOps/SRE engineer is not whether they prevent all failures — it's **how fast they diagnose and fix them** when failures happen. That speed comes from knowing exactly which commands to run in which order, and knowing what the output is telling you.

**Observability** is the ability to understand the internal state of a system from its external outputs — logs, metrics, and events. Without observability, you're flying blind.

---

## Part 1 — The Three Pillars of Observability

```
┌─────────────┬───────────────────────────────────────┬──────────────────────────┐
│  Pillar     │  What It Tells You                    │  Kubernetes Tool         │
├─────────────┼───────────────────────────────────────┼──────────────────────────┤
│  Logs       │  What happened inside your app        │  kubectl logs            │
│  Metrics    │  How much resource is being consumed  │  kubectl top, Prometheus │
│  Traces     │  Which service caused a slow request  │  Jaeger, OpenTelemetry   │
└─────────────┴───────────────────────────────────────┴──────────────────────────┘
```

Today you master the **first two pillars** — the ones you'll use daily as a DevOps engineer — plus the complete troubleshooting methodology for the most common production failures. Distributed tracing (Jaeger, OpenTelemetry) is covered in Phase 6's EKS module where you'll deploy a full observability stack.

---

## Part 2 — Your 7-Command Diagnostic Toolkit

These seven commands solve 90% of all Kubernetes production incidents. Learn them in order — this is the exact sequence you follow when something breaks.

### Command 1 — `kubectl get` — The First Look

Always start here. Get a high-level picture of what's running and what isn't.

```bash
kubectl get pods -n production                            # Are Pods running?
kubectl get pods -n production -o wide                   # Which node? What IP?
kubectl get pods -n production -w                        # Watch for changes live
kubectl get all -n production                            # Everything at once
kubectl get events -n production --sort-by=.lastTimestamp  # Recent cluster events
```

The `-w` flag (watch) is invaluable during incidents — it streams changes to the terminal as they happen, so you can see Pods crashing and restarting in real time.

---

### Command 2 — `kubectl describe` — The Full Story

Once you've spotted a problematic Pod, describe it. The **Events section at the bottom** is where Kubernetes tells you exactly what went wrong.

```bash
kubectl describe pod <pod-name> -n production
```

Always scroll to the bottom. The Events section reveals:
- Image pull failures with the exact error message
- Scheduling failures (why the Pod can't find a node)
- OOMKills (memory limit exceeded)
- Readiness probe failures
- Volume mount failures

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  BackOff           2m    kubelet            Back-off pulling image "myapp:bad-tag"
  Warning  Failed            2m    kubelet            Error: ErrImagePull
  Warning  OOMKilling        5m    kernel             Out of memory: Killed process 1234
```

Every production incident starts with `kubectl describe pod`.

---

### Command 3 — `kubectl logs` — What the App Says

Once you know the Pod exists and has been running, read what it printed to stdout/stderr.

```bash
kubectl logs <pod-name> -n production                           # Current logs
kubectl logs <pod-name> -n production -f                        # Follow live (like tail -f)
kubectl logs <pod-name> -n production --previous                # Logs from BEFORE the crash
kubectl logs <pod-name> -n production --tail=100                # Last 100 lines only
kubectl logs <pod-name> -n production -c <container-name>       # Specific container in multi-container Pod
kubectl logs <pod-name> -n production --since=1h                # Logs from the last hour
```

### The Critical One: `--previous`

When a container crashes and Kubernetes restarts it, the **new container starts with empty logs**. The crash logs are gone from the current container. `--previous` retrieves the logs from the container that just died — this is how you diagnose `CrashLoopBackOff`.

```bash
# Pod is in CrashLoopBackOff
kubectl logs <pod-name> -n production --previous
# Now you can see the stack trace, the error message, the final output
# before the container crashed
```

---

### Command 4 — `kubectl exec` — Get Inside the Container

When logs aren't enough — when you need to verify environment variables, check if a config file was mounted correctly, or test whether the app can reach the database — you exec into the container directly. This is your SSH equivalent for containers.

```bash
kubectl exec -it <pod-name> -n production -- sh             # Open a shell (use bash if sh fails)
kubectl exec -it <pod-name> -n production -- env            # Check all environment variables
kubectl exec -it <pod-name> -n production -- cat /etc/app/config.yaml  # Check mounted config
kubectl exec -it <pod-name> -n production -- df -h          # Check disk mounts
kubectl exec -it <pod-name> -n production -- curl http://database-service:5432  # Test connectivity
kubectl exec -it <pod-name> -n production -- nslookup other-service  # Test DNS resolution
```

When an app behaves unexpectedly, exec in and verify:
- Are the environment variables what you expect?
- Is the config file mounted and does it have the right content?
- Can the container reach the database service? (DNS resolution, port access)
- Is the disk mounted and is there enough space?

---

### Command 5 — `kubectl top` — Resource Consumption

`kubectl top` shows real-time CPU and memory usage (requires Metrics Server running):

```bash
kubectl top pods -n production                              # CPU/memory per Pod
kubectl top pods -n production --sort-by=cpu               # Find the CPU hog
kubectl top pods -n production --sort-by=memory            # Find the memory hog
kubectl top nodes                                          # Node-level consumption
```

> `kubectl top` is useful for immediate inspection but has **no history**. If a memory spike happened 30 minutes ago and resolved, `kubectl top` shows nothing. This is why Prometheus is essential — it records metrics over time.

---

### Command 6 — `kubectl rollout` — Deployment Health

When a deployment is not completing or you need to roll back:

```bash
kubectl rollout status deployment/<name> -n production      # Is this deployment complete?
kubectl rollout history deployment/<name> -n production     # What changed and when?
kubectl rollout undo deployment/<name> -n production        # Emergency rollback to previous version
kubectl rollout undo deployment/<name> --to-revision=3 -n production  # Roll back to specific version
kubectl rollout restart deployment/<name> -n production     # Force a rolling restart
```

`rollout undo` is your emergency brake. When a bad deployment is causing errors, run this immediately — it rolls back to the last known-good version in seconds. Always do this *before* investigating the root cause, so you stop the bleeding first.

---

### Command 7 — `kubectl debug` — Advanced Live Debugging

For containers built with minimal images (distroless, Alpine, scratch) that have no shell:

```bash
# Attach a debug container to a running Pod without restarting it
kubectl debug -it <pod-name> -n production   --image=busybox   --target=<container-name>

# Create a copy of a broken Pod with full debugging tools
kubectl debug <pod-name> -n production   --copy-to=debug-pod   --image=ubuntu   -it -- bash
```

The second variant creates a full copy of the broken Pod but replaces the image with Ubuntu (which has `curl`, `dig`, `netstat`, `strace`, and everything else). You can run it alongside the broken Pod in production without disrupting anything.

---

## Part 3 — Liveness & Readiness Probes

Before covering production failures, you need to understand the health check system — probes are central to several of those failures.

Kubernetes has **no inherent way to know if your app is healthy** just because its process is running. A Java app could be running but deadlocked. A web server could be running but out of database connections. Probes give Kubernetes a way to ask the app directly.

### The Three Probe Types

```yaml
spec:
  containers:
  - name: web-app
    image: my-app:1.0

    livenessProbe:              # "Is this container alive?"
      httpGet:
        path: /health           # Hit this endpoint
        port: 8080
      initialDelaySeconds: 30   # Wait 30s before first check (let the app start)
      periodSeconds: 10         # Check every 10 seconds
      failureThreshold: 3       # Restart after 3 consecutive failures

    readinessProbe:             # "Is this container ready to receive traffic?"
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3       # Remove from Service endpoints after 3 failures

    startupProbe:               # "Has this slow app finished starting?"
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30      # Allow 5 minutes to start (30 attempts × 10s)
      periodSeconds: 10
```

| Probe | On Failure | Use Case |
|---|---|---|
| **Liveness** | Restarts the container | Deadlock — app is running but completely frozen |
| **Readiness** | Removes Pod from Service endpoints | App is starting up or temporarily overloaded |
| **Startup** | Delays liveness/readiness checks | Slow-starting apps like Java/Spring Boot |

### The Production Rule

> **Every production Pod must have both a liveness and a readiness probe.**
>
> Without a readiness probe, Kubernetes routes traffic to Pods the moment the container process starts — before the app has initialized, loaded its config, or connected to the database. Users get errors for the first 10–30 seconds of every deployment.
>
> Without a liveness probe, a deadlocked app continues to receive traffic and return errors indefinitely, since Kubernetes thinks it's healthy (the process is running).

### Probe Methods

| Method | YAML Key | Use When |
|---|---|---|
| HTTP GET | `httpGet` | App exposes an HTTP health endpoint (most common) |
| TCP Socket | `tcpSocket` | App accepts TCP connections but has no HTTP (databases) |
| Exec command | `exec` | Run a command inside the container; exit 0 = healthy |

```yaml
# TCP probe example — for databases
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 15
  periodSeconds: 10
```

---

## Part 4 — The 6 Most Common Production Failures

### Failure 1 — `CrashLoopBackOff`

**What it means:** The container starts, crashes immediately, Kubernetes restarts it, it crashes again — in an exponentially backing-off loop. The delay between restarts grows: 10s → 20s → 40s → 80s → 160s → 300s (capped at 5 minutes).

**Diagnostic sequence:**

```bash
# Step 1: Check the exit code — this tells you WHY it crashed
kubectl describe pod <pod-name> -n production | grep -A10 "Last State"
# Look for:
# Exit Code 1   = application error (check logs for the error message)
# Exit Code 137 = OOMKilled (container exceeded memory limit)
# Exit Code 139 = Segmentation fault
# Exit Code 143 = SIGTERM (graceful termination — usually intentional)

# Step 2: Read the crash logs (the current container has empty logs)
kubectl logs <pod-name> -n production --previous
# This shows what the crashed container was saying before it died
# Look for: stack traces, "error connecting to database", "file not found", etc.

# Step 3: Decode the error and fix
# App error (exit 1)  → fix the application code or configuration
# OOMKill (exit 137)  → increase memory limits
# Wrong env variable  → check ConfigMap/Secret injection
# Missing file        → check volume mounts and PVC binding
```

**Common real-world causes:**
- App can't connect to the database — wrong hostname in environment variable
- App crashes on startup due to missing required environment variable
- App requires a file that isn't mounted (ConfigMap or Secret misconfiguration)
- App exceeds memory limit on startup (set limits too low)

---

### Failure 2 — `ImagePullBackOff` / `ErrImagePull`

**What it means:** Kubernetes cannot pull the container image from the registry.

```bash
# The describe output will show something like:
kubectl describe pod <pod-name> -n production | grep -A5 Events
# Failed to pull image "myapp:v2.0": rpc error: pull access denied for myapp
```

**Common causes and fixes:**

| Cause | What You See | Fix |
|---|---|---|
| Wrong image tag | `manifest unknown: manifest tagged "v2.0" not found` | Correct the tag in the Deployment spec |
| Image doesn't exist | `repository does not exist` | Build and push the image first |
| Private registry, no pull secret | `pull access denied` | Create and reference an `imagePullSecret` |
| Network issue | `timeout`, `connection refused` | Check node network connectivity |

```bash
# Fix for private registry
kubectl create secret docker-registry regcred   --docker-server=<registry-url>   --docker-username=<username>   --docker-password=<password>   -n production

# Then reference it in the Pod spec:
# spec:
#   imagePullSecrets:
#   - name: regcred
#   containers:
#   - name: app
#     image: my-private-registry.com/myapp:v2.0
```

---

### Failure 3 — Pending Pod

**What it means:** The Pod exists but the Scheduler can't find a suitable node to place it on.

```bash
kubectl describe pod <pod-name> -n production | grep -A10 Events
```

The Events section gives you the exact reason — each message maps to a specific fix:

| Message | Root Cause | Fix |
|---|---|---|
| `insufficient cpu` | All nodes are out of CPU capacity | Scale the node group, reduce CPU requests, or add nodes |
| `insufficient memory` | All nodes are out of memory | Same as above |
| `had untolerated taint` | All nodes have taints the Pod doesn't tolerate | Add a toleration or check which node the Pod should go on |
| `had unbound PVCs` | PVC isn't bound to a PV | Fix the PVC/StorageClass issue (covered in Phase 3) |
| `node(s) didn't match Pod's node affinity` | No nodes match the affinity rules | Check node labels vs. affinity expressions |

```bash
# Check node capacity
kubectl describe nodes | grep -A5 "Allocated resources"

# Check existing resource consumption
kubectl top nodes
```

---

### Failure 4 — Service Not Reachable

**What it means:** Your app is running and healthy, but requests to the Service aren't reaching it. The app is unreachable from other Pods or from the load balancer.

```bash
# Step 1: Check if the Service has any endpoints
kubectl get endpoints <service-name> -n production
# If the output shows: <none> → the Service selector doesn't match any Pod labels
# This is the cause of 90% of Service connectivity issues

# Step 2: Compare the Service selector vs. Pod labels
kubectl describe service <service-name> -n production | grep Selector
# Output: Selector: app=my-app,version=v2

kubectl get pods -n production --show-labels
# Output: my-app-xxx   app=my-app,version=v1  ← version label mismatch!

# Step 3: Test connectivity from inside the cluster
kubectl run test --image=busybox --rm -it --restart=Never --   wget -qO- http://<service-name>.<namespace>.svc.cluster.local

# Step 4: Test DNS resolution specifically
kubectl run dns-test --image=busybox --rm -it --restart=Never --   nslookup <service-name>.<namespace>.svc.cluster.local
```

**Fix the label mismatch:**

```bash
kubectl patch service <service-name> -n production   -p '{"spec":{"selector":{"app":"my-app","version":"v2"}}}'

# Immediately verify endpoints populate
kubectl get endpoints <service-name> -n production
# Should now show Pod IPs
```

---

### Failure 5 — `OOMKilled` (Memory Limit Exceeded)

**What it means:** The container's memory usage exceeded the `limits.memory` value in the Pod spec. The Linux kernel forcibly killed the process. Exit code is 137.

```bash
# Confirm OOMKill (look for OOMKilled in the output)
kubectl get pod <pod-name> -n production   -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# Output: OOMKilled

# Check how much memory the Pod is currently using
kubectl top pod <pod-name> -n production

# Check what the current memory limit is
kubectl describe pod <pod-name> -n production | grep -A3 Limits
```

**Fix:**

```bash
# Increase the memory limit
kubectl set resources deployment/<name> -n production   --limits=memory=512Mi   --requests=memory=256Mi

# Watch the rollout
kubectl rollout status deployment/<name> -n production

# Verify the new Pod doesn't get OOMKilled
kubectl top pods -n production -w
```

> **OOMKill vs. node-level OOM:** If the node itself runs out of memory, it applies a `memory-pressure` taint and starts evicting Pods based on their Quality of Service class. Pods without resource limits are evicted first (BestEffort), then Pods where usage exceeds requests (Burstable), then Guaranteed Pods last. This is another reason to always set resource limits.

---

### Failure 6 — Deployment Stuck (Not Progressing)

**What it means:** `kubectl rollout status` hangs, showing "Waiting for deployment to rollout" indefinitely. The deployment is partially updated but some Pods aren't becoming healthy.

```bash
kubectl rollout status deployment/<name> -n production
# Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
# (hangs here)

# Find which Pod is stuck
kubectl get pods -n production | grep -v Running
# NAME                      READY   STATUS              RESTARTS
# webapp-new-xxx            0/1     CrashLoopBackOff    4

# Now diagnose that specific Pod
kubectl logs webapp-new-xxx -n production --previous

# If the new image is bad — rollback immediately
kubectl rollout undo deployment/<name> -n production

# If resource quota is exceeded
kubectl describe resourcequota -n production
```

**Why deployments get stuck:**
- New image has a bug and fails the liveness/readiness probe
- New image requires env vars or secrets that don't exist yet
- ResourceQuota is exceeded — the new Pods can't be created
- New image requests more resources than available on any node

---

## Part 5 — Prometheus + Grafana — The Production Monitoring Stack

`kubectl top` is powerful but ephemeral — it shows only the current moment and has no history. When an incident happens at 3 AM and is resolved by 9 AM, you need to be able to see what happened at 3 AM. That requires **time-series metrics storage**.

The standard production stack on Kubernetes:

```
Every Pod/Node
    ↓  exposes metrics at /metrics endpoint (Prometheus format)
Prometheus (scrapes every 15s, stores in time-series DB)
    ↓  queries (PromQL)
Grafana (dashboards, alerting rules)
    ↓  sends alerts
AlertManager → Slack / PagerDuty / email
```

### Installing kube-prometheus-stack

On EKS (Phase 6), you install the entire stack with a single Helm command:

```bash
# Add the Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the full stack: Prometheus + Grafana + AlertManager + node-exporter
helm install prometheus prometheus-community/kube-prometheus-stack   --namespace monitoring   --create-namespace   --set grafana.adminPassword=YourPassword

# Check all components are running
kubectl get pods -n monitoring
```

This single command gives you:
- Prometheus pre-configured to scrape all cluster metrics (CPU, memory, network, disk)
- Grafana with pre-built Kubernetes dashboards (Node Overview, Pod Overview, Cluster Overview)
- AlertManager configured to route alerts
- Node Exporter on every node for OS-level metrics
- kube-state-metrics for Kubernetes object metrics (replicas, pod states, etc.)

### Accessing Grafana

```bash
# Port-forward Grafana to your local machine
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Open browser: http://localhost:3000
# Default credentials: admin / YourPassword (from the install command)
```

### Essential Grafana Dashboards

Once inside Grafana, the most useful pre-built dashboards are:
- **Dashboard 15760** — Kubernetes Cluster Overview (nodes, namespaces, resource utilization)
- **Dashboard 6417** — Kubernetes Pod Overview (per-pod CPU, memory, restarts)
- **Dashboard 1860** — Node Exporter Full (OS-level: disk I/O, network, CPU steal)

### Writing a Basic Alert Rule

```yaml
# Alert: Pod restarting more than 5 times in 10 minutes
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-restart-alert
  namespace: monitoring
spec:
  groups:
  - name: pod.rules
    rules:
    - alert: PodFrequentlyRestarting
      expr: |
        increase(kube_pod_container_status_restarts_total[10m]) > 5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is restarting frequently"
        description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has restarted more than 5 times in the last 10 minutes"
```

---

## Part 6 — The Full Diagnostic Loop

When something breaks in production, follow this exact sequence. Don't skip steps.

```
Step 1: kubectl get pods -n <namespace>
         ↓ Find the unhealthy Pod(s)

Step 2: kubectl describe pod <pod-name> -n <namespace>
         ↓ Read the Events section — this usually tells you everything

Step 3: kubectl logs <pod-name> -n <namespace> --previous
         ↓ Read what the crashed container was saying before death

Step 4: kubectl exec -it <pod-name> -n <namespace> -- env
         ↓ Verify environment variables are correct

Step 5: kubectl get endpoints <svc-name> -n <namespace>
         ↓ Verify Service routing is correct (if unreachable)

Step 6: kubectl top pods -n <namespace>
         ↓ Check for resource pressure

Step 7: kubectl rollout undo deployment/<name> -n <namespace>
         ↓ If bad deployment — rollback first, diagnose second
```

> The first thing you do in **any** production incident is `kubectl describe pod` + scroll to Events. 90% of the time, Kubernetes tells you exactly what's wrong right there.

---

## Part 7 — Hands-On Lab: Project 10

**Platform:** KodeKloud Kubernetes playground
**Time estimate:** 60 minutes
**Goal:** Deploy five broken applications, diagnose all five using only kubectl commands, fix each one, and practice the full diagnostic loop

### Step 1 — Deploy Five Broken Apps

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: debug-lab
---
# App 1: Wrong image tag → ImagePullBackOff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-broken-image
  namespace: debug-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: broken-image
  template:
    metadata:
      labels:
        app: broken-image
    spec:
      containers:
      - name: app
        image: nginx:nonexistent-tag-999
---
# App 2: Service selector mismatch → Service not reachable
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-bad-selector
  namespace: debug-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: bad-selector-svc
  namespace: debug-lab
spec:
  selector:
    app: frontend   # WRONG — Pods are labeled app=backend
  ports:
  - port: 80
    targetPort: 80
---
# App 3: Impossible resource requests → Pending forever
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-no-resources
  namespace: debug-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hungry
  template:
    metadata:
      labels:
        app: hungry
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "99"         # 99 full CPU cores — no node has this
            memory: "999Gi"
---
# App 4: Missing ConfigMap → CrashLoopBackOff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-bad-config
  namespace: debug-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bad-config
  template:
    metadata:
      labels:
        app: bad-config
    spec:
      containers:
      - name: app
        image: nginx:1.25
        envFrom:
        - configMapRef:
            name: nonexistent-config   # This ConfigMap does not exist
---
# App 5: Healthy control — confirm your tools work
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-healthy
  namespace: debug-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: healthy
  template:
    metadata:
      labels:
        app: healthy
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
EOF
```

### Step 2 — Your Challenge: Diagnose All Five

Use only these commands — no looking at the YAML above:

```bash
kubectl get pods -n debug-lab
kubectl describe pod <name> -n debug-lab
kubectl logs <name> -n debug-lab
kubectl get endpoints -n debug-lab
kubectl top pods -n debug-lab
```

For each broken app, identify:
1. What is the **symptom**? (`Pending`, `ImagePullBackOff`, `CrashLoopBackOff`, `0/1 Running`, etc.)
2. What is the **root cause**?
3. What is the **exact fix**?

Expected answers before you check:

| App | Symptom | Root Cause | Fix |
|---|---|---|---|
| `app-broken-image` | `ImagePullBackOff` | Image tag doesn't exist | Set correct image tag |
| `app-bad-selector` | Running but service has `<none>` endpoints | Service selector `app=frontend` doesn't match Pod labels `app=backend` | Patch Service selector |
| `app-no-resources` | `Pending` | Requests 99 CPUs — impossible | Set realistic resource requests |
| `app-bad-config` | `CreateContainerConfigError` | ConfigMap `nonexistent-config` doesn't exist | Create the ConfigMap |
| `app-healthy` | `Running` ✅ | — | Nothing needed |

### Step 3 — Fix Each One

```bash
# Fix 1: Correct the image tag
kubectl set image deployment/app-broken-image app=nginx:1.25 -n debug-lab

# Fix 2: Correct the Service selector to match Pod labels
kubectl patch service bad-selector-svc -n debug-lab   -p '{"spec":{"selector":{"app":"backend"}}}'
kubectl get endpoints bad-selector-svc -n debug-lab   # Should now show Pod IPs

# Fix 3: Fix the impossible resource request
kubectl set resources deployment/app-no-resources -n debug-lab   --requests=cpu=200m,memory=128Mi   --limits=cpu=500m,memory=256Mi

# Fix 4: Create the missing ConfigMap
kubectl create configmap nonexistent-config   --from-literal=KEY=value   -n debug-lab
kubectl rollout restart deployment/app-bad-config -n debug-lab
```

### Step 4 — Verify All Five Are Running

```bash
kubectl get pods -n debug-lab
# All five Deployments should show Running status ✅
```

### Step 5 — Practice the Full Diagnostic Loop

For each app that was broken, run through the complete sequence from scratch and practice reading the output without already knowing the answer:

```bash
# For each pod that was broken:
kubectl describe pod <pod-name> -n debug-lab | tail -20   # Events section
kubectl logs <pod-name> -n debug-lab --previous 2>/dev/null || echo "No crash logs"
kubectl get endpoints -n debug-lab
```

### Step 6 — Add Probes to the Healthy App

Practice writing probes on the healthy app:

```bash
kubectl patch deployment app-healthy -n debug-lab --patch '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "app",
          "livenessProbe": {
            "httpGet": {"path": "/", "port": 80},
            "initialDelaySeconds": 10,
            "periodSeconds": 10,
            "failureThreshold": 3
          },
          "readinessProbe": {
            "httpGet": {"path": "/", "port": 80},
            "initialDelaySeconds": 5,
            "periodSeconds": 5,
            "failureThreshold": 3
          }
        }]
      }
    }
  }
}'

# Watch the rollout apply the new probe config
kubectl rollout status deployment/app-healthy -n debug-lab

# Describe a pod and confirm probes are registered
kubectl describe pod -l app=healthy -n debug-lab | grep -A8 "Liveness\|Readiness"
```

### Step 7 — Clean Up

```bash
kubectl delete namespace debug-lab
```

---

## Essential kubectl for This Phase

```bash
# First-look commands
kubectl get pods -n <ns> -o wide                          # Status + node + IP
kubectl get events -n <ns> --sort-by=.lastTimestamp       # Recent events newest-last
kubectl get all -n <ns>                                   # All resources at once

# Deep inspection
kubectl describe pod <name> -n <ns>                       # Full story with Events
kubectl describe node <name>                              # Node health, capacity, events
kubectl describe service <name> -n <ns>                   # Service config + endpoints

# Logs
kubectl logs <pod> -n <ns>                                # Current logs
kubectl logs <pod> -n <ns> --previous                     # Pre-crash logs
kubectl logs <pod> -n <ns> -f                             # Live stream
kubectl logs <pod> -n <ns> --tail=50 --since=30m          # Last 50 lines, last 30 min
kubectl logs <pod> -n <ns> -c <container>                 # Multi-container Pod

# Exec
kubectl exec -it <pod> -n <ns> -- sh                      # Shell into container
kubectl exec -it <pod> -n <ns> -- env | grep DB           # Check specific env vars
kubectl exec -it <pod> -n <ns> -- curl http://other-svc   # Test connectivity

# Rollout management
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>
kubectl rollout undo deployment/<name> -n <ns>            # Rollback
kubectl rollout restart deployment/<name> -n <ns>         # Force restart

# Resource usage
kubectl top pods -n <ns> --sort-by=memory
kubectl top nodes

# Endpoints check
kubectl get endpoints <svc-name> -n <ns>                  # Must show Pod IPs, not <none>
```

---

## Phase 5 Checklist

- [ ] Understand the 3 pillars of observability: Logs, Metrics, Traces
- [ ] Use `kubectl get pods -o wide` to identify unhealthy Pods
- [ ] Use `kubectl describe pod` and read the Events section to diagnose failures
- [ ] Use `kubectl logs --previous` to read crash logs from a CrashLoopBackOff Pod
- [ ] Use `kubectl exec` to verify environment variables and connectivity inside a container
- [ ] Use `kubectl top` to find CPU/memory hogs
- [ ] Use `kubectl rollout undo` to perform an emergency rollback
- [ ] Use `kubectl debug` to attach a debug container to a minimal image Pod
- [ ] Configure a liveness probe, readiness probe, and startup probe
- [ ] Diagnose and fix an `ImagePullBackOff` from the Events output
- [ ] Diagnose and fix a `CrashLoopBackOff` using `--previous` logs and exit codes
- [ ] Diagnose and fix a Pending Pod from the Events output
- [ ] Diagnose and fix a Service with `<none>` endpoints
- [ ] Diagnose and fix an OOMKilled Pod by increasing memory limits
- [ ] Perform an emergency rollback on a stuck deployment
- [ ] Understand why Prometheus is needed beyond `kubectl top`
- [ ] Know the Prometheus + Grafana + AlertManager architecture

---

## What's Next — Phase 6 Preview

Phase 6 is the **production phase** — where everything built in Phases 1–5 becomes enterprise-grade:

- **Helm** — package manager for Kubernetes; bundle all your YAML into versioned, parameterized charts deployable with a single command
- **RBAC** — Role-Based Access Control; define exactly who can do what inside the cluster
- **Amazon EKS** — running Kubernetes on AWS with managed control plane, IAM integration, and production-grade networking
- **ArgoCD & GitOps** — Git as the single source of truth; every push to your repo automatically deploys to the cluster

---

*Phase 5 of 6 — Kubernetes Learning Series | Mustafa Kanaan*
