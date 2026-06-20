# Phase 2 — Core Workloads & Networking

> **Learning Path:** Zero → Production-Ready | **Phase:** 2 of 6
> **Prerequisite:** Phase 1 — Architecture & Foundations complete

---

## Why Not Just Use Bare Pods?

A bare Pod has one critical weakness: **it is not self-healing**. If the node it runs on dies, the Pod is gone permanently — Kubernetes will not reschedule it. Nobody runs bare Pods in production.

**Workload controllers** are higher-level objects that manage Pods on your behalf. You define desired state; the controller creates and maintains the Pods.

---

## Part 1 — Workload Controllers

### Deployments

A **Deployment** is the most common workload controller. It manages a **ReplicaSet**, which manages Pods. You almost never interact with the ReplicaSet directly.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Allow 1 extra Pod during update
      maxUnavailable: 0  # Never go below 3 healthy Pods
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### Rolling Updates & Rollbacks

When you update an image, Kubernetes replaces Pods gradually. New Pods become healthy before old ones are terminated — zero downtime.

```bash
# Trigger a rolling update
kubectl set image deployment/web-app app=nginx:1.26

# Watch rollout progress
kubectl rollout status deployment/web-app

# View rollout history
kubectl rollout history deployment/web-app

# Rollback to previous version
kubectl rollout undo deployment/web-app

# Rollback to a specific revision
kubectl rollout undo deployment/web-app --to-revision=2
```

> **Production rule:** Always set `maxUnavailable: 0` for critical services. Zero downtime during updates at the cost of slightly more resources during the rollout window.

---

### StatefulSets

A **StatefulSet** is for workloads that require **stable identity** — databases, message queues, distributed caches. Unlike Deployments, StatefulSets guarantee:

- **Stable, persistent Pod names:** `postgres-0`, `postgres-1`, `postgres-2`
- **Ordered startup and shutdown:** Pod 0 must be Running before Pod 1 starts
- **Stable network identity:** each Pod gets its own DNS hostname
- **Persistent volume per Pod:** each replica gets its own PVC that survives Pod deletion

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"   # Required: headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random hash (`app-7d4b9c-xyz`) | Stable ordinal (`app-0`, `app-1`) |
| Pod startup order | Parallel | Sequential |
| Storage | Usually shared | Per-Pod PVC |
| Use case | Stateless apps, web servers | Databases, queues, distributed systems |
| DNS hostname per Pod | No | Yes |

---

### DaemonSets

A **DaemonSet** ensures exactly **one Pod runs on every node** (or a subset). When new nodes join, the DaemonSet schedules on them automatically.

**Common production use cases:**
- Log collection agents (Fluentd, Filebeat)
- Node monitoring (Prometheus Node Exporter, Datadog)
- Network plugins (Calico, Flannel CNI agents)
- Security agents

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd:v1.16
        resources:
          limits:
            memory: "200Mi"
            cpu: "100m"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

---

### Jobs and CronJobs

**Jobs** run a Pod to completion — for batch tasks, data migrations, or one-time operations. A Job ensures the task runs to success, retrying on failure.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3           # Retry up to 3 times on failure
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: myapp:latest
        command: ["python", "manage.py", "migrate"]
```

**CronJobs** schedule Jobs on a cron expression:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"      # Every night at 02:00
  concurrencyPolicy: Forbid  # Don't start new job if previous still running
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["sh", "-c", "backup.sh"]
```

---

## Part 2 — Resource Management

### Requests vs. Limits

This is one of the most important concepts in production Kubernetes and the most common source of incidents.

| | Request | Limit |
|---|---|---|
| Definition | What the container is **guaranteed** to get | The **maximum** the container can use |
| Used by scheduler | ✅ Yes — decides which node to place Pod on | ❌ No |
| CPU over-limit behavior | Container is **throttled** (slowed, keeps running) | Container is **throttled** |
| Memory over-limit behavior | Not killed | Container is **OOM-killed immediately** |

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "250m"     # 250 millicores = 0.25 of one CPU core
  limits:
    memory: "256Mi"
    cpu: "500m"
```

> **CPU units:** `1000m` = 1 full CPU core. `250m` = 25% of one core.
> **Memory units:** `Mi` = mebibytes. `Gi` = gibibytes.

---

### ⚠️ Production Issue: The Noisy Neighbor

**Problem:** A developer sets `requests.memory: 50Mi` but forgets to set `limits.memory`. The app has a memory leak and grows to 4GB, starving every other Pod on the node. Multiple unrelated Pods get OOM-killed. This is a **noisy neighbor incident**.

**Solution:** Always set both requests AND limits on every production workload. Enforce this cluster-wide with a `LimitRange`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      memory: "256Mi"
      cpu: "500m"
    defaultRequest:
      memory: "64Mi"
      cpu: "100m"
    max:
      memory: "2Gi"
      cpu: "2000m"
```

### ResourceQuotas — Namespace-Level Guardrails

`ResourceQuota` caps total resources a namespace can consume:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "10"
```

---

## Part 3 — Health Checks (Probes)

Kubernetes uses three types of probes to determine container health.

| Probe | Question it answers | Action on failure |
|---|---|---|
| **Liveness** | Is the container still alive? Should it restart? | Container is **restarted** |
| **Readiness** | Is the container ready to receive traffic? | Pod is **removed from Service endpoints** |
| **Startup** | Has the app finished initial startup? | Liveness/Readiness paused until this passes |

```yaml
containers:
- name: app
  image: myapp:1.0
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 15
    failureThreshold: 3       # Restart after 3 consecutive failures

  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10
    failureThreshold: 3       # Remove from load balancer after 3 failures

  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30      # Allow up to 300 seconds for startup (30 x 10s)
    periodSeconds: 10
```

---

### ⚠️ Production Issue: Aggressive Liveness Probe Causes Restart Loop

**Problem:** A liveness probe with `failureThreshold: 1` and `periodSeconds: 5` restarts a Java app every time it takes more than 5 seconds to respond during a GC pause or traffic spike. The app enters **CrashLoopBackOff** under normal load.

**Solution:**
- Use `startupProbe` to give slow-starting apps time to initialize
- Set conservative `failureThreshold` values (3–5)
- Keep liveness probes **lightweight** — never hit the database from a liveness probe
- Use **readiness** (not liveness) when the app is temporarily overloaded — remove it from traffic, don't kill it

---

## Part 4 — Kubernetes Networking

### The Networking Model

Kubernetes enforces a flat networking model with three rules:
1. Every Pod gets its own unique IP address
2. All Pods can communicate with all other Pods **without NAT**
3. All nodes can communicate with all Pods **without NAT**

This is enabled by a **CNI plugin** (e.g., Calico, Flannel, Cilium) installed in the cluster.

### The Problem: Pod IPs Are Ephemeral

Every time a Pod dies and is replaced, it gets a **completely different IP address**. If your frontend hardcodes the backend's Pod IP, that connection breaks on every restart.

**Kubernetes solves this with Services.**

---

## Services

A **Service** is a stable virtual IP address and DNS name that always points to the correct set of Pods, no matter how many times those Pods restart.

The Service uses a **label selector** to find its target Pods. `kube-proxy` on every node updates routing rules (iptables/IPVS) in real time as Pods come and go.

### The Three Service Types

#### ClusterIP (Default)
Internal-only. Accessible only from within the cluster. Use for backend services, databases, anything that should never be exposed externally.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-service
spec:
  type: ClusterIP
  selector:
    app: payments
  ports:
  - port: 80          # Port the Service listens on
    targetPort: 3000  # Port the container actually listens on
```

#### NodePort
Opens a specific port (30000–32767) on **every node**. External traffic reaches the app via `<NodeIP>:<NodePort>`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080    # Optional — auto-assigned if omitted
```

> **Production note:** NodePort exposes node IPs publicly, has no TLS termination, no hostname routing. Not recommended for production external access — use Ingress instead.

#### LoadBalancer
Provisions a **cloud load balancer** automatically (AWS ALB/NLB, GCP LB). Standard way to expose a service externally on managed Kubernetes.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
  - port: 443
    targetPort: 8443
```

> **Cost warning:** Each `LoadBalancer` Service provisions a separate cloud load balancer. 20 services = 20 load balancers. This is expensive — which is why **Ingress** exists.

---

### Understanding the Three Port Fields

This confuses most engineers initially:

```yaml
ports:
- port: 80          # Port the SERVICE listens on (what callers use)
  targetPort: 3000  # Port the CONTAINER is listening on
  nodePort: 30080   # (NodePort only) Port opened on every Node
```

**Real-world example:** Your Node.js app listens on `3000` (developer habit), but company standard says all HTTP services must be on port `80`. Solution: `port: 80`, `targetPort: 3000`. Callers use `:80`; the Service translates to `:3000` on the Pod.

> **Debug tip:** `kubectl describe service <name>` shows exact port mappings and resolved Endpoints. If Endpoints shows `<none>`, your label selector doesn't match any running Pods.

---

### How Services Work Under the Hood

1. You create a Service with a selector
2. Kubernetes creates an **Endpoints** object listing IPs of all matching Pods
3. `kube-proxy` on every node watches Endpoints and writes **iptables/IPVS** rules
4. When traffic hits the Service ClusterIP, rules redirect it to a healthy Pod IP

```bash
# See live Pod IPs behind a Service
kubectl get endpoints payments-service

# NAME               ENDPOINTS                                         AGE
# payments-service   10.244.1.5:3000,10.244.2.3:3000,10.244.3.7:3000  5d
```

### DNS Inside the Cluster

Every Service gets a DNS name managed by **CoreDNS**:

```
<service-name>.<namespace>.svc.cluster.local
```

From within the same namespace, the short form works:
```bash
# Full DNS name
curl http://payments-service.production.svc.cluster.local:80

# Short form (same namespace)
curl http://payments-service:80
```

---

## Ingress

**Ingress** manages **external HTTP/HTTPS routing** to Services. Instead of one cloud load balancer per Service, you have **one load balancer** (the Ingress Controller) routing to many Services.

```
Internet
   |
   ▼
[Ingress Controller]  ← single external LoadBalancer Service
   |
   ├── api.myapp.com       → api-service:80
   ├── myapp.com/          → frontend-service:80
   └── admin.myapp.com     → admin-service:8080
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls-secret
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 3000
```

### Service Types at a Glance

| | ClusterIP | NodePort | LoadBalancer | Ingress |
|---|---|---|---|---|
| External access | ❌ | ✅ (node IP) | ✅ (cloud LB) | ✅ (cloud LB) |
| TLS termination | ❌ | ❌ | Partial | ✅ |
| Host-based routing | ❌ | ❌ | ❌ | ✅ |
| Path-based routing | ❌ | ❌ | ❌ | ✅ |
| Cloud LB cost | None | None | 1 per Service | 1 total |
| Recommended for | Internal traffic | Dev/testing | Single service | Multiple services |

---

## NetworkPolicy — Pod-Level Firewall Rules

By default, all Pods can talk to all other Pods. In production, this is a security risk. **NetworkPolicy** defines firewall rules at the Pod level.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-payments
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payments
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway   # Only api-gateway can reach payments
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres      # payments can only talk to postgres
    ports:
    - protocol: TCP
      port: 5432
```

> **Important:** NetworkPolicy requires a CNI plugin that supports it (Calico, Cilium). The default CNI in many environments does NOT enforce NetworkPolicy — verify your CNI before relying on these rules in production.

---

## Common Production Issues

### ⚠️ Issue 1: Service Returns No Endpoints

**Symptom:** curl to a Service hangs or connection refused. `kubectl get endpoints <service>` shows `<none>`.

**Root cause:** Service selector does not match any running Pod labels.

```bash
# Check what labels the Service is selecting for
kubectl describe service my-service

# Check what labels Pods actually have
kubectl get pods --show-labels

# The selector must be a subset of the Pod labels
```

**Fix:** Correct either the Service selector or the Pod labels to match.

---

### ⚠️ Issue 2: CrashLoopBackOff

**Symptom:** Pod keeps restarting. Status shows `CrashLoopBackOff`.

**Root cause options:**
- App crashing on startup (bad config, missing env var, DB connection refused)
- Liveness probe too aggressive
- OOM killed (memory limit too low)

```bash
# See logs from before the crash
kubectl logs <pod-name> --previous

# See events and exit code
kubectl describe pod <pod-name>

# Exit code 137 = OOM killed  → increase memory limit
# Exit code 1   = app error   → check logs
# Exit code 2   = shell error → check command syntax
```

---

### ⚠️ Issue 3: Pod Stuck in Pending

**Symptom:** Pod stays in `Pending` state indefinitely.

**Root cause options:**
- No node has enough CPU/memory to satisfy resource requests
- Node selector or affinity rules cannot be satisfied
- PersistentVolumeClaim not bound

```bash
kubectl describe pod <pending-pod-name>
# Look at the "Events" section at the bottom
# Common: "0/3 nodes are available: 3 Insufficient memory"
```

**Fix options:** Reduce resource requests, add nodes, or fix affinity rules.

---

## Essential kubectl for This Phase

```bash
# Deployment management
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl rollout status deployment/web-app
kubectl rollout history deployment/web-app
kubectl rollout undo deployment/web-app
kubectl scale deployment web-app --replicas=5

# Services and networking
kubectl get services
kubectl get endpoints
kubectl describe service my-service

# Debugging
kubectl get pods -o wide                          # Shows which node each Pod is on
kubectl describe pod <pod-name>                   # Full events — always check first
kubectl logs <pod-name>                           # Current logs
kubectl logs <pod-name> --previous                # Logs before last restart
kubectl exec -it <pod-name> -- /bin/sh            # Shell into container
kubectl top pods                                  # CPU/memory usage
kubectl top nodes

# Local testing
kubectl port-forward service/my-service 8080:80  # Access service at localhost:8080
```

---

## Phase 2 Checklist

- [ ] Understand why bare Pods are not used in production
- [ ] Write and deploy a Deployment with a rolling update strategy
- [ ] Perform a rolling update and rollback using `kubectl rollout`
- [ ] Understand when to use StatefulSet vs. Deployment
- [ ] Understand the purpose of DaemonSets
- [ ] Set resource requests and limits on all containers
- [ ] Write liveness and readiness probes for an application
- [ ] Understand ClusterIP, NodePort, and LoadBalancer differences
- [ ] Understand what Ingress is and when to use it
- [ ] Debug a Service with no Endpoints
- [ ] Diagnose CrashLoopBackOff and OOMKilled containers

---

*Phase 2 of 6 — Kubernetes Learning Series | Mustafa Kanaan*
