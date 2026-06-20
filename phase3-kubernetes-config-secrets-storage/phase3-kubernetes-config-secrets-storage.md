# Phase 3 — Configuration, Secrets & Persistent Storage

> **Learning Path:** Zero → Production-Ready | **Phase:** 3 of 6
> **Prerequisite:** Phase 2 — Workloads & Networking complete

---

## Why This Phase Exists

By the end of Phase 2, you could deploy apps, expose them with Services, and route traffic with Ingress. But every app you deployed had a critical flaw: configuration was either hardcoded into the image or missing entirely.

Real production apps need three things beyond just running:
1. **Configuration** — database hostnames, ports, feature flags, log levels
2. **Secrets** — passwords, API keys, TLS certificates, tokens
3. **Persistent storage** — a place to write data that survives Pod restarts

This phase solves all three.

---

## Part 1 — The Problem With Hardcoded Configuration

Imagine you build a Docker image for your payment service. Inside the image you hardcode:

```
DB_HOST: prod-mysql.internal
DB_PASSWORD: TurSup3r$ecret
LOG_LEVEL: WARN
```

Now your QA team needs the same app pointing to a **test database**. You build a second image. The password rotates — you rebuild again. You need a debug build with `LOG_LEVEL: DEBUG` — third image. You now have three images of the same app differing only in configuration. This is a maintenance disaster. And if that image ever gets pushed to a public registry by mistake, your production database password is exposed.

**The rule in production:** An image is **immutable and environment-agnostic**. It should contain zero configuration. Configuration is injected at runtime, depending on where the container runs.

Kubernetes solves this with two objects:
- **ConfigMap** — for non-sensitive configuration
- **Secret** — for sensitive configuration

---

## Part 2 — ConfigMaps

A **ConfigMap** is a key-value store that lives in Kubernetes and gets injected into your Pod at runtime. The Pod has no idea whether the values came from a ConfigMap or were hardcoded — it just sees environment variables or files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DB_HOST: "prod-mysql.production.svc.cluster.local"
  DB_PORT: "5432"
  LOG_LEVEL: "WARN"
  MAX_CONNECTIONS: "100"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    feature_flags:
      new_checkout: true
```

Notice two things:
- Simple key-value pairs (`LOG_LEVEL: "WARN"`) are injected as environment variables
- Multi-line values with a `|` (like `config.yaml`) are injected as full files

### Method 1 — Inject All Keys as Environment Variables

```yaml
containers:
- name: app
  image: myapp:1.0
  envFrom:
  - configMapRef:
      name: app-config    # Every key in the ConfigMap becomes an env var
```

### Method 2 — Inject Specific Keys

```yaml
containers:
- name: app
  image: myapp:1.0
  env:
  - name: DATABASE_HOST        # Name the env var sees
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DB_HOST           # Key inside the ConfigMap
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
```

### Method 3 — Inject as a Mounted File

Some apps expect a config file at a specific path (Nginx, Prometheus, custom tools). Mount the ConfigMap as a volume:

```yaml
containers:
- name: app
  image: myapp:1.0
  volumeMounts:
  - name: config-volume
    mountPath: /etc/app          # config.yaml will appear at /etc/app/config.yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
```

### A Critical Operational Difference

> **When you update a ConfigMap:**
> - Environment variables are **NOT** updated automatically — the Pod must restart to see new values
> - Mounted files **ARE** updated automatically (with a ~60s propagation delay)
>
> This difference causes real production incidents when engineers update a ConfigMap expecting the app to immediately pick up new values, but the Pod is still using the old environment variables.

### When to Use Which Method

| Method | Best For |
|---|---|
| `envFrom` (all keys) | Simple apps that read all config from env vars |
| `env` (specific keys) | When you need to rename keys or inject selectively |
| Mounted files | Apps that expect a config file path (Nginx, Prometheus, etc.) |

---

## Part 3 — Secrets

A **Secret** works identically to a ConfigMap in how it's injected — same two methods (env vars and mounted files). But with two critical differences:

1. Values are **base64-encoded** at rest in etcd
2. Kubernetes treats them differently — they can be **encrypted at rest** via KMS, restricted via RBAC, and are never printed in plain text in logs

### ⚠️ The Most Important Misconception About Secrets

> **base64 is NOT encryption. It is encoding.**
> Anyone can decode it in one command:
> ```bash
> echo "VHVyU3VwM3IkZWNyZXQ=" | base64 --decode
> # TurSup3r$ecret
> ```
> The real security of Secrets comes from two things:
> 1. **etcd encryption at rest** — AWS EKS does this automatically with KMS
> 2. **RBAC policies** — restricting which ServiceAccounts can `get` or `list` Secrets

### Creating a Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: production
type: Opaque
data:
  # Values MUST be base64-encoded
  # Run: echo -n "TurSup3r$ecret" | base64
  password: VHVyU3VwM3IkZWNyZXQ=
  username: cG9zdGdyZXM=
```

Or create it imperatively — Kubernetes base64-encodes for you:

```bash
kubectl create secret generic postgres-secret   --from-literal=username=postgres   --from-literal=password=TurSup3r$ecret   -n production
```

### Injecting a Secret into a Pod

```yaml
containers:
- name: app
  image: myapp:1.0
  env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: password
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: username
```

Or mount as a file (the right pattern for TLS certificates and SSH keys):

```yaml
containers:
- name: app
  volumeMounts:
  - name: tls-certs
    mountPath: /etc/ssl/certs
    readOnly: true
volumes:
- name: tls-certs
  secret:
    secretName: myapp-tls-secret
```

### ConfigMap vs. Secret

| | ConfigMap | Secret |
|---|---|---|
| Use for | Non-sensitive config | Passwords, API keys, TLS certs, tokens |
| Stored in etcd as | Plain text | base64-encoded (encryptable with KMS) |
| Printed in plain text in logs | Yes | No |
| AWS equivalent | SSM Parameter Store (standard) | AWS Secrets Manager / SSM SecureString |

---

## Part 4 — Persistent Storage

### The Stateless vs. Stateful Problem

Everything in Phase 2 was **stateless**. If an Nginx Pod dies and gets replaced, nothing is lost — the new Pod starts fresh and works perfectly. Stateless is easy.

Now imagine that Pod is a **MySQL database**. It stores customer orders, payment history, user accounts. The Pod crashes. Kubernetes replaces it with a new Pod — with a **completely empty filesystem**. All your data is gone.

This is the stateless vs. stateful problem, and it's the single biggest architectural challenge when running databases in Kubernetes.

The solution is **Persistent Volumes** — storage that exists independently of any Pod, and can be re-attached when a Pod is replaced.

---

### The Four Storage Layers

Kubernetes splits storage into four distinct layers. Understanding *why* each exists matters more than memorizing the YAML.

```
┌─────────────────────────────────────────────────────────┐
│  Layer 4 — VolumeMount (inside the Pod spec)            │
│  Maps a claimed volume to a path in the container       │
├─────────────────────────────────────────────────────────┤
│  Layer 3 — PersistentVolumeClaim (PVC)                  │
│  A storage request: "I need 10Gi, ReadWriteOnce"        │
├─────────────────────────────────────────────────────────┤
│  Layer 2 — PersistentVolume (PV)                        │
│  The actual storage: an EBS volume, NFS share, disk     │
├─────────────────────────────────────────────────────────┤
│  Layer 1 — StorageClass                                 │
│  Defines how to automatically create PVs on demand      │
└─────────────────────────────────────────────────────────┘
```

The separation exists so that **developers** only think about what they need (PVC), while **infrastructure** manages the actual storage (PV + StorageClass). A developer requests "10Gi of fast SSD storage" without caring whether it's an AWS EBS volume, a GCP persistent disk, or an NFS share.

---

### PersistentVolume (PV)

A **PV** is a piece of actual storage that exists in your infrastructure — an AWS EBS volume, an NFS share, a local disk. It exists independently of any Pod and has its own lifecycle.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce         # Only one node can mount read/write at a time
  persistentVolumeReclaimPolicy: Retain   # Keep the data even if the PVC is deleted
  storageClassName: gp3
  awsElasticBlockStore:
    volumeID: vol-0a1b2c3d4e5f6g7h8
    fsType: ext4
```

**Access Modes — the most commonly confused concept in storage:**

| Mode | Short | Meaning | Typical Use |
|---|---|---|---|
| `ReadWriteOnce` | RWO | One node mounts read/write | Databases, stateful apps |
| `ReadOnlyMany` | ROX | Many nodes mount read-only | Shared config, static assets |
| `ReadWriteMany` | RWX | Many nodes mount read/write | Shared file systems (NFS, EFS) |

> **Important:** RWO means one **node**, not one Pod. Two Pods on the same node can both mount an RWO volume — but this is rare and usually a mistake for databases.

---

### PersistentVolumeClaim (PVC)

A **PVC** is a storage request submitted by a developer. It says "I need 20Gi of ReadWriteOnce storage." Kubernetes finds a matching PV and **binds** them together. Once bound, that PV is exclusively reserved for that PVC.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: gp3
```

Using the PVC in a Pod:

```yaml
containers:
- name: postgres
  image: postgres:15
  volumeMounts:
  - name: data
    mountPath: /var/lib/postgresql/data   # Data written here survives Pod restarts
volumes:
- name: data
  persistentVolumeClaim:
    claimName: postgres-pvc
```

The PVC is the bridge between the Pod and the actual storage. If the Pod is deleted and replaced, the new Pod mounts the **same PVC** and finds all the data exactly where it was left.

---

### StorageClass — Dynamic Provisioning

Without a StorageClass, a cluster admin must manually pre-create every PV before a developer can claim it. This doesn't scale. A **StorageClass** tells Kubernetes how to **automatically create PVs on demand** the moment a PVC is submitted.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # Default for all PVCs
provisioner: ebs.csi.aws.com       # AWS EBS CSI driver creates the volumes
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"                # Encrypt the EBS volume
reclaimPolicy: Delete              # Delete the EBS volume when PVC is deleted
volumeBindingMode: WaitForFirstConsumer   # ← Critical on AWS (explained below)
allowVolumeExpansion: true
```

### Why `WaitForFirstConsumer` Is Critical on AWS

AWS EBS volumes are **Availability Zone-specific**. If you create a PVC without this setting, Kubernetes immediately provisions an EBS volume — but picks a random AZ. If your Pod then gets scheduled on a node in a different AZ, it can never mount the volume and stays stuck in `Pending` forever.

`WaitForFirstConsumer` tells Kubernetes: "Don't create the EBS volume yet. Wait until the Pod is scheduled to a node, then create the volume in that node's AZ."

```bash
# Always verify your StorageClass has this setting on AWS
kubectl describe storageclass gp3 | grep VolumeBinding
# VolumeBindingMode: WaitForFirstConsumer  ← correct
```

---

### The Complete Storage Flow

```
Developer submits a PVC (requests 20Gi gp3)
          ↓
StorageClass provisioner triggers
          ↓
Pod gets scheduled to node in us-east-1a
          ↓
EBS volume created in us-east-1a (same AZ as the node)
          ↓
PV object created automatically in Kubernetes
          ↓
PVC and PV are Bound to each other
          ↓
Pod mounts the PVC → data lives at /var/lib/postgresql/data
          ↓
Pod crashes, Kubernetes creates a replacement Pod
          ↓
Replacement Pod mounts the SAME PVC → all data is still there ✅
```

---

### volumeClaimTemplates — Automatic Per-Pod PVCs for StatefulSets

In Phase 2 you learned that StatefulSets give each Pod a stable, predictable identity (`mysql-0`, `mysql-1`, `mysql-2`). That stable identity also applies to storage.

If you run a database with 3 replicas, each replica needs its **own** dedicated volume — not a shared one. `volumeClaimTemplates` tells the StatefulSet to automatically create a separate PVC for each Pod, bound permanently to that Pod's identity.

```yaml
# Inside a StatefulSet spec:
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: gp3
    resources:
      requests:
        storage: 20Gi
```

This automatically creates and permanently binds:
- `data-mysql-0` PVC → `mysql-0` Pod
- `data-mysql-1` PVC → `mysql-1` Pod
- `data-mysql-2` PVC → `mysql-2` Pod

If `mysql-1` crashes and restarts, it always reconnects to `data-mysql-1`. The binding is permanent — Kubernetes never reassigns a StatefulSet PVC to a different Pod.

> **Why not just use a regular Deployment with a single PVC for a database?**
> - Deployment Pod names are random hashes — they change on restart
> - All replicas would compete for the same PVC (impossible with RWO access mode)
> - There's no ordered, stable identity for replica sets to coordinate primary/secondary roles
>
> **Always use StatefulSet + `volumeClaimTemplates` for any stateful workload.**

---

## Part 5 — Production Issues

### ⚠️ Issue 1: Secret Baked Into a Docker Image

**Scenario:** A developer hardcodes `DB_PASSWORD=TurSup3r$ecret` in the Dockerfile or app code. The image is pushed to a container registry. A security audit finds the password baked into a layer. The image was accidentally pushed to a public registry — the production database password is now public.

**Why it happens:** Developers take shortcuts during development and forget to clean up before pushing.

**The fix:**
- **Never** commit secrets to Git or bake them into images
- Use Kubernetes Secrets injected at runtime as env vars or mounted files
- Rotate secrets without rebuilding images: update the Secret object, restart the Pod, done
- On EKS: use **IAM Roles for Service Accounts (IRSA)** to let Pods fetch secrets directly from AWS Secrets Manager — no Secret objects in Kubernetes at all

---

### ⚠️ Issue 2: Secret Committed to Git

**Scenario:** A developer runs `kubectl get secret postgres-secret -o yaml > secret.yaml` and commits it to the team's Git repository. The base64-encoded secret is now in version control forever — even after deletion, it may exist in forks, CI caches, or git history.

**The fix:**
- Never commit Secret YAML to Git
- Use **Sealed Secrets** (Bitnami) or **External Secrets Operator** + AWS Secrets Manager / HashiCorp Vault for GitOps-safe secret management
- Add `*.secret.yaml` to `.gitignore` and enforce with pre-commit hooks
- On EKS: IRSA eliminates the need for most Secret objects entirely

---

### ⚠️ Issue 3: Pod Stuck in Pending — Unbound PVC

**Scenario:** You deploy a StatefulSet or Pod with a PVC. The Pod stays in `Pending` indefinitely. `kubectl describe pod` shows:

```
0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims
```

**Root causes:**
1. No StorageClass exists or the `storageClassName` in the PVC doesn't match any StorageClass
2. On KodeKloud/local playgrounds — no default StorageClass with dynamic provisioning
3. `WaitForFirstConsumer` + no node available in the required AZ

**Diagnosis:**
```bash
# Step 1: Check if a StorageClass exists with a provisioner
kubectl get storageclass

# Step 2: Check PVC status — is it Bound or Pending?
kubectl get pvc -n <namespace>

# Step 3: Read the exact failure reason
kubectl describe pvc <pvc-name> -n <namespace>
# Look at Events at the bottom
```

**Fix on KodeKloud/local environments** (no cloud provisioner available):

```yaml
# Create a PV manually using hostPath for local testing
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /mnt/data    # Uses a directory on the node (local only — never production)
```

---

### ⚠️ Issue 4: EBS Volume Created in the Wrong Availability Zone

**Scenario:** A PVC is submitted. Kubernetes immediately provisions an EBS volume in `us-east-1a`. The Pod is scheduled on a node in `us-east-1b`. The Pod stays `Pending` forever because EBS volumes cannot be mounted across AZs.

**Diagnosis:**
```bash
kubectl describe pod <pod-name> | grep -A5 Events
# You'll see: 
# had volume node affinity conflict
```

**Fix:** Always set `volumeBindingMode: WaitForFirstConsumer` in your StorageClass. If you inherited a cluster without this setting, you'll need to delete the PVC, update the StorageClass, and resubmit.

---

### ⚠️ Issue 5: ConfigMap Update Not Reflected in Running Pod

**Scenario:** You update a ConfigMap with a new `LOG_LEVEL` or database host. The running Pods keep using the old value. You're confused because the ConfigMap clearly shows the new value.

**Root cause:** Environment variables are snapshot at Pod startup. Updating a ConfigMap does **not** push new values to running containers that use `env:` or `envFrom:`. Only mounted-file ConfigMaps auto-update.

**Fix:**
```bash
# Force a rolling restart to pick up new ConfigMap values
kubectl rollout restart deployment/<name> -n <namespace>

# Verify the new value is live inside the container
kubectl exec -it <pod-name> -n <namespace> -- env | grep LOG_LEVEL
```

---

## Part 6 — Hands-On Lab: Project 6 & 7

**Platform:** KodeKloud Kubernetes playground
**Time estimate:** 45 minutes
**Goal:** Deploy a complete app stack that reads all config from ConfigMaps and Secrets, stores data on a PVC

### Step 1 — Create the ConfigMap and Secret

```bash
# ConfigMap for non-sensitive config
kubectl create configmap app-config   --from-literal=DB_HOST=postgres.production.svc.cluster.local   --from-literal=DB_PORT=5432   --from-literal=LOG_LEVEL=WARN   --from-literal=APP_ENV=production   -n production

# Secret for sensitive config
kubectl create secret generic app-secret   --from-literal=DB_PASSWORD=TurSup3r$ecret   --from-literal=API_KEY=sk-prod-key-abc123   -n production

# Verify
kubectl get configmap app-config -n production -o yaml
kubectl get secret app-secret -n production -o yaml
```

### Step 2 — Decode a Secret Value (Verify It's Correct)

```bash
kubectl get secret app-secret -n production   -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
# Output: TurSup3r$ecret
```

### Step 3 — Create the PVC

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
EOF
```

```bash
# Watch the PVC — it should go from Pending to Bound
kubectl get pvc -n production -w
```

### Step 4 — Deploy the App Using ConfigMap, Secret, and PVC

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: nginx:1.25
        envFrom:
        - configMapRef:
            name: app-config        # All ConfigMap keys as env vars
        env:
        - name: DB_PASSWORD         # Individual Secret keys
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: API_KEY
        volumeMounts:
        - name: data
          mountPath: /var/app/data  # Persistent storage for app data
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data-pvc
EOF
```

### Step 5 — Verify Everything Is Wired Correctly

```bash
# Get a running Pod name
kubectl get pods -n production

# Verify env vars are injected correctly
kubectl exec -it <pod-name> -n production -- env | grep -E "DB_HOST|LOG_LEVEL|DB_PASSWORD"

# Verify the PVC is mounted
kubectl exec -it <pod-name> -n production -- df -h | grep /var/app/data

# Write a file to persistent storage
kubectl exec -it <pod-name> -n production -- sh -c "echo 'test data' > /var/app/data/test.txt"

# Delete the Pod — Kubernetes will replace it
kubectl delete pod <pod-name> -n production

# Exec into the NEW Pod — the file should still be there
kubectl get pods -n production   # Wait for new Pod to be Running
kubectl exec -it <new-pod-name> -n production -- cat /var/app/data/test.txt
# Output: test data  ← proves data survived the Pod restart ✅
```

### Step 6 — Update the ConfigMap and See the Behavior

```bash
# Update the ConfigMap
kubectl patch configmap app-config -n production   --patch '{"data":{"LOG_LEVEL":"DEBUG"}}'

# Check the env var inside the running Pod — it will still show WARN
kubectl exec -it <pod-name> -n production -- env | grep LOG_LEVEL
# Output: LOG_LEVEL=WARN  ← old value, Pod hasn't restarted yet

# Force a rolling restart
kubectl rollout restart deployment/webapp -n production

# Now check again
kubectl exec -it <new-pod-name> -n production -- env | grep LOG_LEVEL
# Output: LOG_LEVEL=DEBUG  ← new value ✅
```

### Step 7 — Clean Up

```bash
kubectl delete deployment webapp -n production
kubectl delete pvc app-data-pvc -n production
kubectl delete configmap app-config -n production
kubectl delete secret app-secret -n production
```

---

## Essential kubectl for This Phase

```bash
# ConfigMaps
kubectl create configmap <name> --from-literal=KEY=VALUE
kubectl create configmap <name> --from-file=config.yaml
kubectl get configmap <name> -n <namespace> -o yaml
kubectl describe configmap <name> -n <namespace>
kubectl edit configmap <name> -n <namespace>

# Secrets
kubectl create secret generic <name> --from-literal=KEY=VALUE
kubectl get secret <name> -n <namespace> -o yaml
kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 --decode
kubectl describe secret <name> -n <namespace>     # Shows keys but NOT values

# Storage
kubectl get pv                                    # List all PersistentVolumes
kubectl get pvc -n <namespace>                    # List PVCs in a namespace
kubectl get storageclass                          # List StorageClasses
kubectl describe pvc <name> -n <namespace>        # See binding status and Events
kubectl describe pv <name>                        # See PV details

# Verify what's inside a container
kubectl exec -it <pod-name> -n <namespace> -- env           # Check env vars
kubectl exec -it <pod-name> -n <namespace> -- df -h         # Check disk mounts
kubectl exec -it <pod-name> -n <namespace> -- ls /etc/app   # Check mounted files
```

---

## Phase 3 Checklist

- [ ] Understand why code and config must be completely separated
- [ ] Create a ConfigMap and inject it using `envFrom`
- [ ] Create a ConfigMap and inject it using `env` (specific keys)
- [ ] Mount a ConfigMap as a file inside a container
- [ ] Understand the env var vs. mounted file update behavior difference
- [ ] Create a Secret imperatively with `kubectl create secret`
- [ ] Decode a Secret value with `base64 --decode`
- [ ] Understand that base64 is encoding — not encryption
- [ ] Understand the four storage layers: StorageClass → PV → PVC → VolumeMount
- [ ] Understand the difference between RWO, ROX, and RWX access modes
- [ ] Create a PVC and attach it to a Deployment
- [ ] Prove data survives a Pod restart using a PVC
- [ ] Understand `volumeClaimTemplates` in StatefulSets (per-Pod dedicated PVCs)
- [ ] Understand why `WaitForFirstConsumer` matters on AWS EKS
- [ ] Debug a PVC stuck in Pending state

---

## What's Next — Phase 4 Preview

Phase 4 covers **Scheduling, Taints & Autoscaling** — how Kubernetes decides which node a Pod runs on, and how the cluster scales itself:

- **Taints & Tolerations** — marking nodes as off-limits for certain workloads
- **Node Affinity** — attracting Pods to specific nodes based on labels
- **Resource Quotas & LimitRanges** — namespace-level resource enforcement
- **HPA (Horizontal Pod Autoscaler)** — automatically scaling Pods based on CPU/memory
- **Load testing with k6** — proving your HPA works under real traffic

---

*Phase 3 of 6 — Kubernetes Learning Series | Mustafa Kanaan*
