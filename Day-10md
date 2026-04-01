# 📘 Chapter 10 — Resource Quotas, Limits & Probes

> This chapter separates juniors from seniors. Knowing how to **constrain resources** and **define health checks** properly is what keeps production clusters stable. Heavily asked in interviews.

---

## 🔷 Why This Matters

```
Without resource limits:

  App A has a memory leak
       ↓
  Consumes all 16GB RAM on node
       ↓
  Node runs out of memory
       ↓
  K8s starts killing OTHER pods on same node
       ↓
  Your entire team's apps go down 💀

Without probes:

  App starts but is still loading
       ↓
  K8s sends traffic to it immediately
       ↓
  Users get 502 errors ❌

  OR

  App deadlocks silently — process running
  but not responding to requests
       ↓
  K8s thinks it's healthy (process is up)
       ↓
  Traffic keeps hitting dead app 💀
```

---

## 🔷 PART 1 — Resource Requests & Limits

### Requests vs Limits

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  requests  = minimum guaranteed resources               │
│              "I need at least this much to run"         │
│              Used by SCHEDULER to place pod on a node   │
│                                                         │
│  limits    = maximum allowed resources                  │
│              "I must never exceed this"                 │
│              Enforced by KUBELET at runtime             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**CPU behavior:**
```
CPU request → scheduler finds node with enough CPU
CPU limit   → container is THROTTLED (slowed down)
              never killed for CPU — just slowed
```

**Memory behavior:**
```
Memory request → scheduler finds node with enough memory
Memory limit   → container is KILLED (OOMKilled)
                 if it exceeds limit
                 → pod restarts
```

---

### 🔷 CPU & Memory Units

```
CPU:
  1 CPU    = 1000m (millicores)
  0.5 CPU  = 500m
  0.25 CPU = 250m

  250m  → 25% of one CPU core
  1000m → one full CPU core
  2000m → two full CPU cores

Memory:
  Ki = Kibibyte  (1024 bytes)    ← k8s uses this
  Mi = Mebibyte  (1024 Ki)
  Gi = Gibibyte  (1024 Mi)

  64Mi  → 64 Mebibytes
  128Mi → 128 Mebibytes
  1Gi   → 1 Gibibyte
```

---

### 🛠️ Resource Requests & Limits in Pod YAML

```yaml
# resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
  namespace: dev
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          memory: "64Mi"     # scheduler: find node with 64Mi free
          cpu: "250m"        # scheduler: find node with 0.25 CPU free
        limits:
          memory: "128Mi"    # kill if exceeds 128Mi (OOMKilled)
          cpu: "500m"        # throttle if exceeds 500m
```

---

### 🔷 What Happens Without Requests/Limits?

```
No requests set:
  Scheduler has no info → places pod anywhere
  Node might be overloaded → pod runs slowly
  "Burstable" QoS class → first to be evicted

No limits set:
  Pod can consume unlimited resources
  One bad pod can starve every other pod on node
  Memory leak = node crash

Production rule:
  ALWAYS set both requests AND limits ✅
```

---

### 🔷 QoS Classes (Quality of Service)

K8s assigns QoS class automatically based on requests/limits:

```
┌───────────────┬──────────────────────────────┬───────────────┐
│  QoS Class    │  Condition                   │  Eviction     │
├───────────────┼──────────────────────────────┼───────────────┤
│ Guaranteed    │ requests == limits           │ Last evicted  │
│               │ (both set, same value)       │ (safest)      │
├───────────────┼──────────────────────────────┼───────────────┤
│ Burstable     │ requests < limits            │ Middle        │
│               │ (most common setup)          │               │
├───────────────┼──────────────────────────────┼───────────────┤
│ BestEffort    │ no requests or limits set    │ First evicted │
│               │                              │ (most risky)  │
└───────────────┴──────────────────────────────┴───────────────┘
```

```bash
# Check QoS class of a pod
kubectl describe pod resource-demo -n dev | grep "QoS Class"
# QoS Class: Burstable
```

---

## 🔷 PART 2 — LimitRange

### What is LimitRange?

Sets **default** requests/limits for all pods in a namespace
that don't specify their own.

```
Without LimitRange:
  Dev forgets to set resources in their pod YAML
  → BestEffort QoS → first evicted → app crashes

With LimitRange:
  Admin sets namespace defaults
  → every pod gets sensible defaults automatically
```

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limitrange
  namespace: dev
spec:
  limits:
    - type: Container
      default:                    # applied if no limits set
        memory: "256Mi"
        cpu: "500m"
      defaultRequest:             # applied if no requests set
        memory: "128Mi"
        cpu: "250m"
      max:                        # no container can exceed this
        memory: "1Gi"
        cpu: "2000m"
      min:                        # no container can go below this
        memory: "64Mi"
        cpu: "100m"
```

```bash
kubectl apply -f limitrange.yaml

# Now any pod in 'dev' namespace without resources
# automatically gets defaults ✅

kubectl describe limitrange dev-limitrange -n dev
```

---

## 🔷 PART 3 — ResourceQuota

### What is ResourceQuota?

LimitRange = per container defaults
ResourceQuota = **total budget for entire namespace**

```
Team A gets namespace 'team-a':
  Total CPU:    max 10 cores
  Total Memory: max 20Gi
  Total Pods:   max 20
  Total PVCs:   max 10

Team B gets namespace 'team-b':
  Total CPU:    max 4 cores
  Total Memory: max 8Gi
  Total Pods:   max 10

Teams are isolated — one team can't steal
resources from another ✅
```

```yaml
# resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    # Compute resources
    requests.cpu: "4"              # total CPU requests in namespace
    requests.memory: "8Gi"        # total memory requests
    limits.cpu: "8"               # total CPU limits
    limits.memory: "16Gi"         # total memory limits

    # Object count limits
    pods: "20"                    # max 20 pods
    services: "10"                # max 10 services
    persistentvolumeclaims: "10"  # max 10 PVCs
    secrets: "20"                 # max 20 secrets
    configmaps: "20"              # max 20 configmaps

    # Storage
    requests.storage: "50Gi"      # total storage requested
```

```bash
kubectl apply -f resourcequota.yaml

# Check quota usage
kubectl get resourcequota -n dev
kubectl describe resourcequota dev-quota -n dev

# Output:
# Resource                Used    Hard
# --------                ----    ----
# limits.cpu              2       8
# limits.memory           4Gi     16Gi
# pods                    8       20
# requests.cpu            1       4
# requests.memory         2Gi     8Gi
# services                3       10
```

---

### 🔷 ResourceQuota Enforcement

```bash
# Try to create pod that exceeds quota
kubectl run test-pod --image=nginx \
  --requests=cpu=5 -n dev

# Error:
# Error from server (Forbidden):
# exceeded quota: dev-quota,
# requested: requests.cpu=5,
# used: requests.cpu=3,
# limited: requests.cpu=4  ❌

# This is exactly the protection you want
```

---

## 🔷 PART 4 — Probes

### The 3 Types of Probes

```
┌──────────────────┬──────────────────────────────────────────┐
│  Liveness Probe  │ Is the app ALIVE?                        │
│                  │ Fails → container RESTARTED              │
├──────────────────┼──────────────────────────────────────────┤
│  Readiness Probe │ Is the app READY for traffic?            │
│                  │ Fails → removed from Service endpoints   │
│                  │ (no restart — just stops getting traffic)│
├──────────────────┼──────────────────────────────────────────┤
│  Startup Probe   │ Has the app FINISHED starting?           │
│                  │ Disables liveness/readiness until done   │
│                  │ For slow-starting apps                   │
└──────────────────┴──────────────────────────────────────────┘
```

---

### 🔷 Probe Mechanisms

All 3 probe types can use any of these mechanisms:

```
httpGet     → HTTP GET request to container
              Success: 200-399 status code
              Failure: 4xx/5xx or no response

tcpSocket   → TCP connection attempt
              Success: connection established
              Failure: connection refused

exec        → Run a command inside container
              Success: exit code 0
              Failure: non-zero exit code

grpc        → gRPC health check
              For gRPC-based services
```

---

### 🛠️ Liveness Probe

> "Is this container still working? Or is it stuck/deadlocked?"

```yaml
# liveness-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
  namespace: dev
spec:
  containers:
    - name: app
      image: my-app:latest
      ports:
        - containerPort: 8080

      livenessProbe:
        httpGet:
          path: /healthz          # health endpoint in your app
          port: 8080
        initialDelaySeconds: 15   # wait 15s before first check
                                  # (let app start up)
        periodSeconds: 20         # check every 20 seconds
        failureThreshold: 3       # fail 3 times → restart container
        successThreshold: 1       # 1 success = considered alive
        timeoutSeconds: 5         # wait max 5s for response
```

```
Timeline:
  t=0s:  Container starts
  t=15s: First liveness check → /healthz → 200 OK ✅
  t=35s: Check → 200 OK ✅
  t=55s: Check → 500 Error ❌ (fail 1/3)
  t=75s: Check → 500 Error ❌ (fail 2/3)
  t=95s: Check → 500 Error ❌ (fail 3/3)
         → Container RESTARTED 🔄
  t=110s: First check on restarted container...
```

---

**Liveness with exec:**
```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - "redis-cli ping | grep PONG"  # exit 0 if redis is up
  initialDelaySeconds: 10
  periodSeconds: 15
```

**Liveness with TCP:**
```yaml
livenessProbe:
  tcpSocket:
    port: 3306          # can we connect to MySQL port?
  initialDelaySeconds: 15
  periodSeconds: 20
```

---

### 🛠️ Readiness Probe

> "Is this container ready to receive traffic?"

```yaml
# readiness-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
  namespace: dev
spec:
  containers:
    - name: app
      image: my-app:latest

      readinessProbe:
        httpGet:
          path: /ready             # readiness endpoint
          port: 8080
        initialDelaySeconds: 5    # check sooner than liveness
        periodSeconds: 10
        failureThreshold: 3       # fail 3 times → remove from LB
        successThreshold: 2       # need 2 successes to add back to LB
```

```
What happens when readiness fails:

  Pod is Running ✅ (not restarted)
  But removed from Service Endpoints ❌

  Service: backend-service
    Endpoints BEFORE fail: 10.244.1.5, 10.244.1.6, 10.244.1.7
    Endpoints AFTER fail:  10.244.1.5, 10.244.1.7
                           ↑ 10.244.1.6 removed — no traffic sent to it

  Pod recovers → readiness passes again
    Endpoints: 10.244.1.5, 10.244.1.6, 10.244.1.7 ✅
```

---

### 🔷 Liveness vs Readiness — The Key Difference

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  LIVENESS fails  → container RESTARTED                     │
│                    "This is broken, reboot it"             │
│                                                             │
│  READINESS fails → pod removed from Service                │
│                    container NOT restarted                  │
│                    "Not ready yet, don't send traffic"      │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Real example:
  App is loading large config file on startup (takes 30 seconds)

  ❌ Liveness probe → restarts app every 30s because it's "failing"
     → infinite restart loop → app never starts

  ✅ Readiness probe → holds traffic back for 30s
     → app finishes loading → starts receiving traffic
```

---

### 🛠️ Startup Probe

> For apps that take a long time to start (Java, legacy apps)

```yaml
# startup-probe.yaml
spec:
  containers:
    - name: legacy-java-app
      image: legacy-app:latest

      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30     # allow up to 30 failures
        periodSeconds: 10        # check every 10s
        # = allows up to 30 × 10 = 300 seconds (5 min) to start

      livenessProbe:             # only activates AFTER startup probe passes
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 20
        failureThreshold: 3

      readinessProbe:            # only activates AFTER startup probe passes
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 10
```

```
Without startupProbe:
  Java app takes 3 minutes to start
  Liveness probe set to fail after 1 minute
  → App killed before it finishes starting
  → Infinite CrashLoopBackOff 💀

With startupProbe:
  Liveness/Readiness disabled until startup probe passes
  App gets full 5 minutes to start
  After startup passes → normal probes take over ✅
```

---

### 🛠️ Full Production Pod — Everything Together

```yaml
# production-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
    spec:
      containers:
        - name: app
          image: myapp:v2.1
          ports:
            - containerPort: 8080

          # Resources — always set these
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"

          # Startup — for slow starters
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 12
            periodSeconds: 5        # 60 seconds to start

          # Liveness — restart if broken
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 0  # startup probe handles this
            periodSeconds: 15
            failureThreshold: 3
            timeoutSeconds: 3

          # Readiness — hold traffic until ready
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3

          # Config from ConfigMap & Secret
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secret
```

---

## 🔷 What Your App Needs to Implement

For probes to work, your app must expose health endpoints:

```python
# Django / Python example
from django.http import JsonResponse

def healthz(request):
    # liveness — is app alive?
    return JsonResponse({"status": "ok"}, status=200)

def ready(request):
    # readiness — can we handle traffic?
    try:
        # check DB connection
        from django.db import connection
        connection.ensure_connection()
        return JsonResponse({"status": "ready"}, status=200)
    except Exception:
        return JsonResponse({"status": "not ready"}, status=503)
```

```javascript
// Express / Node.js example
app.get('/healthz', (req, res) => {
  res.status(200).json({ status: 'ok' })
})

app.get('/ready', async (req, res) => {
  try {
    await db.ping()              // check DB
    await redis.ping()           // check cache
    res.status(200).json({ status: 'ready' })
  } catch (err) {
    res.status(503).json({ status: 'not ready', error: err.message })
  }
})
```

---

## ✅ Chapter 10 Summary

| Concept | Key Point |
|---|---|
| requests | Min guaranteed resources — used by scheduler |
| limits | Max allowed — CPU throttled, memory OOMKilled |
| LimitRange | Default requests/limits per namespace |
| ResourceQuota | Total resource budget for entire namespace |
| Liveness Probe | Is app alive? Fails → container restarted |
| Readiness Probe | Is app ready? Fails → removed from Service endpoints |
| Startup Probe | Has app finished starting? Blocks liveness/readiness |
| httpGet | Most common probe mechanism — hit health endpoint |
| OOMKilled | Container exceeded memory limit — killed by k8s |
| QoS Guaranteed | requests == limits — safest, last to be evicted |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What's the difference between requests and limits?"**
   → Requests are the minimum guaranteed resources used by the scheduler to place pods. Limits are the maximum — CPU is throttled at the limit, memory causes OOMKill if exceeded.

2. **"What is the difference between Liveness and Readiness probes?"**
   → Liveness failure restarts the container — for deadlocks or crashes. Readiness failure removes the pod from Service endpoints without restarting — for temporary unavailability like loading data or waiting for a dependency.

3. **"What is a Startup Probe and when do you need it?"**
   → Delays liveness and readiness probes until the app finishes starting. Needed for slow-starting apps like Java services — prevents premature liveness failures causing infinite restart loops.

4. **"What is ResourceQuota and why is it important?"**
   → Sets a total resource budget for a namespace. Prevents one team or app from consuming all cluster resources, enforces fair resource sharing across teams.

5. **"What happens when a pod exceeds its memory limit?"**
   → It gets OOMKilled — the container is terminated immediately. K8s restarts it. If this keeps happening you'll see OOMKilled in pod status and need to either fix the memory leak or increase the limit.

---

Ready for **Chapter 11 → Taints, Tolerations & Node Affinity** — how to control exactly **which pods run on which nodes**? Critical for prod clusters with GPU nodes, spot instances, and dedicated workloads. Say **"next"** 🚀
