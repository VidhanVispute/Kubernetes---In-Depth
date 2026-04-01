# 📘 Chapter 16 — Sidecar & Init Containers

> These are **design patterns** — not just features. Understanding when and why to use them separates engineers who write k8s YAML from engineers who **architect production systems.**

---

## 🔷 The Problem First

```
Real production apps need more than just the app container:

  App needs logs shipped to central store
  → Can't modify app code to add Fluentd
  → Solution: run Fluentd alongside as Sidecar ✅

  App needs TLS termination
  → Don't want to bake Envoy into every app image
  → Solution: Envoy as Sidecar ✅

  App needs config files pulled from Vault before starting
  → App shouldn't start until config is ready
  → Solution: Init Container that runs first ✅

  DB migration must run before app starts
  → Can't let app boot with old schema
  → Solution: Init Container runs migration first ✅
```

---

## 🔷 Multi-Container Pod Patterns

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Sidecar      → helper runs ALONGSIDE main container        │
│                 same lifetime as main container             │
│                 enhances or extends main app                │
│                                                             │
│  Init Container → runs BEFORE main container starts        │
│                   must complete successfully first          │
│                   sets up environment for main app          │
│                                                             │
│  Ambassador   → proxy for outbound traffic (Sidecar type)  │
│                                                             │
│  Adapter      → transforms output of main app              │
│                 (normalizes logs/metrics format)            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔷 PART 1 — Init Containers

### What is an Init Container?

```
Regular pod lifecycle:
  All containers start together → run forever

Pod with Init Containers:
  Init-1 runs → must complete (exit 0)
       ↓
  Init-2 runs → must complete (exit 0)
       ↓
  Init-3 runs → must complete (exit 0)
       ↓
  Main containers start → run forever

Key rules:
  → Init containers run ONE AT A TIME in order
  → Each MUST succeed before next starts
  → If init container fails → pod restarts it
  → Main containers DON'T start until ALL inits complete
  → Init containers have their own image
     (can use tools not in main image)
```

---

### 🔷 Init Container Use Cases

```
1. Wait for dependency to be ready
   "Don't start until MySQL is accepting connections"

2. Database migrations
   "Run alembic upgrade head before app starts"

3. Pull secrets/config from Vault
   "Fetch credentials, write to shared volume"

4. Clone a git repo
   "Pull latest config files before app starts"

5. Set permissions on shared volumes
   "chmod 755 /data before app user can write"

6. Register with service discovery
   "Register this instance before accepting traffic"
```

---

### 🛠️ Init Container YAML

**Example 1 — Wait for database to be ready:**

```yaml
# init-wait-for-db.yaml
apiVersion: v1
kind: Pod
metadata:
  name: django-app
  namespace: dev
spec:
  initContainers:

    # Init-1: Wait for MySQL to be ready
    - name: wait-for-mysql
      image: busybox:1.35
      command:
        - sh
        - -c
        - |
          echo "Waiting for MySQL..."
          until nc -z mysql-service 3306; do
            echo "MySQL not ready yet — sleeping 2s"
            sleep 2
          done
          echo "MySQL is ready! ✅"

    # Init-2: Run database migrations
    - name: run-migrations
      image: myapp/django:latest       # same image as main app
      command:
        - sh
        - -c
        - |
          echo "Running DB migrations..."
          python manage.py migrate --noinput
          echo "Migrations complete! ✅"
      env:
        - name: DB_HOST
          value: "mysql-service"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD

  # Main container only starts after BOTH inits succeed
  containers:
    - name: django
      image: myapp/django:latest
      ports:
        - containerPort: 8000
      env:
        - name: DB_HOST
          value: "mysql-service"
```

```bash
kubectl apply -f init-wait-for-db.yaml

# Watch pod lifecycle
kubectl get pods -n dev -w
# NAME         READY   STATUS             RESTARTS
# django-app   0/1     Init:0/2           0    ← init 1 running
# django-app   0/1     Init:1/2           0    ← init 2 running
# django-app   0/1     PodInitializing    0    ← both done, main starting
# django-app   1/1     Running            0    ← app running ✅

# Logs from init container
kubectl logs django-app -c wait-for-mysql -n dev
# Waiting for MySQL...
# MySQL not ready yet — sleeping 2s
# MySQL not ready yet — sleeping 2s
# MySQL is ready! ✅

kubectl logs django-app -c run-migrations -n dev
# Running DB migrations...
# Migrations complete! ✅
```

---

**Example 2 — Pull config from Vault, share via volume:**

```yaml
# init-vault.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-vault-config
  namespace: dev
spec:
  volumes:
    - name: config-vol               # shared between init and main
      emptyDir: {}                   # lives for pod lifetime

  initContainers:
    - name: vault-init
      image: vault:latest
      command:
        - sh
        - -c
        - |
          vault login $VAULT_TOKEN
          vault kv get -field=config secret/myapp \
            > /config/app-config.json
          echo "Config fetched from Vault ✅"
      env:
        - name: VAULT_ADDR
          value: "http://vault-service:8200"
        - name: VAULT_TOKEN
          valueFrom:
            secretKeyRef:
              name: vault-token
              key: token
      volumeMounts:
        - name: config-vol
          mountPath: /config          # writes config here

  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config      # reads config from here
          readOnly: true
```

---

**Example 3 — Clone git repo for config:**

```yaml
initContainers:
  - name: git-clone
    image: alpine/git:latest
    command:
      - git
      - clone
      - https://github.com/myorg/app-config.git
      - /config
    volumeMounts:
      - name: config-vol
        mountPath: /config

containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
      - name: config-vol
        mountPath: /etc/nginx/conf.d   # uses cloned nginx configs
```

---

### 🔷 Init Container vs Regular Container

```
┌─────────────────────┬─────────────────────┬──────────────────────┐
│                     │  Init Container     │  Regular Container   │
├─────────────────────┼─────────────────────┼──────────────────────┤
│ When it runs        │ Before main         │ After all inits done │
│ Run order           │ Sequential          │ Parallel             │
│ Must complete?      │ Yes (exit 0)        │ Runs forever         │
│ Restart on failure  │ Yes (whole pod)     │ Based on policy      │
│ Probes              │ No                  │ Yes                  │
│ Shares volumes      │ Yes                 │ Yes                  │
│ Shares network      │ Yes                 │ Yes                  │
│ Different image OK? │ Yes ✅              │ Yes ✅               │
└─────────────────────┴─────────────────────┴──────────────────────┘
```

---

## 🔷 PART 2 — Sidecar Containers

### What is a Sidecar?

```
Sidecar = helper container that runs ALONGSIDE main container
          in the same pod — forever — same lifecycle

┌──────────────────────────────────────────────────────┐
│                       POD                            │
│                                                      │
│  ┌─────────────────┐    ┌─────────────────┐          │
│  │   Main App      │    │    Sidecar      │          │
│  │   Container     │    │    Container    │          │
│  │                 │    │                 │          │
│  │  Writes logs    │───▶│  Ships logs     │          │
│  │  to /var/log    │    │  to Elasticsearch│         │
│  │                 │    │                 │          │
│  └─────────────────┘    └─────────────────┘          │
│                                                      │
│  Shared: Network (localhost), Volumes               │
└──────────────────────────────────────────────────────┘
```

**Why sidecar instead of baking into main image?**
```
Separation of concerns:
  → App team owns app container
  → Platform team owns sidecar (logging, security, proxy)
  → Update logging without touching app ✅
  → Same sidecar pattern across all services ✅
  → App stays simple and focused ✅
```

---

### 🔷 Sidecar Use Cases

```
1. Log shipping
   App writes to file → Sidecar ships to Elasticsearch

2. Metrics scraping
   App exposes /metrics → Sidecar exposes to Prometheus

3. Proxy / Service mesh
   All traffic flows through Envoy sidecar
   (this is exactly how Istio works)

4. TLS termination
   Sidecar handles HTTPS → forwards HTTP to app

5. Config refresh
   Sidecar watches ConfigMap → reloads app config on change

6. Secret rotation
   Sidecar watches for new secrets → updates app

7. Compression / encryption
   Sidecar compresses data before writing
```

---

### 🛠️ Sidecar Example 1 — Log Shipping

```yaml
# sidecar-logging.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-log-sidecar
  namespace: dev
spec:
  volumes:
    - name: log-vol
      emptyDir: {}               # shared log volume

  containers:
    # Main application
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app    # app writes logs here

    # Sidecar: ships logs to Elasticsearch
    - name: log-shipper
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app    # reads app logs
          readOnly: true
      env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch-service"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
      resources:
        requests:
          cpu: "50m"
          memory: "50Mi"
        limits:
          cpu: "100m"
          memory: "100Mi"
```

---

### 🛠️ Sidecar Example 2 — Nginx + Config Reloader

```yaml
# sidecar-config-reload.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-reloader
  namespace: dev
spec:
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-config         # ConfigMap with nginx.conf

  containers:
    # Main: nginx
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d

    # Sidecar: watches ConfigMap, reloads nginx on change
    - name: config-reloader
      image: jimmidyson/configmap-reload:latest
      args:
        - --volume-dir=/etc/nginx/conf.d
        - --webhook-url=http://localhost:80/-/reload
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
```

---

### 🛠️ Sidecar Example 3 — Metrics Adapter

```yaml
# sidecar-metrics.yaml
apiVersion: v1
kind: Pod
metadata:
  name: legacy-app-with-metrics
  namespace: dev
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9102"
spec:
  containers:
    # Main: legacy app that exposes proprietary metrics
    - name: legacy-app
      image: legacy-app:latest
      ports:
        - containerPort: 8080        # app's proprietary metrics port

    # Sidecar: converts proprietary format → Prometheus format
    - name: metrics-adapter
      image: prom/statsd-exporter:latest
      ports:
        - containerPort: 9102        # Prometheus scrapes this
      args:
        - --statsd.listen-udp=:8125
        - --web.listen-address=:9102
```

---

### 🔷 Sidecar Pattern — Communication Methods

```
Sidecars in same pod share:

1. NETWORK (localhost)
   Main app:  listens on localhost:8080
   Sidecar:   calls localhost:8080
   → No network hop, ultra low latency

2. VOLUMES (shared filesystem)
   Main app:  writes to /var/log/app.log
   Sidecar:   reads from /var/log/app.log
   → Simple file-based communication

3. PROCESS SIGNALS (via shared PID namespace)
   Sidecar:   sends SIGHUP to nginx process
   Nginx:     reloads config
   → Requires shareProcessNamespace: true
```

```yaml
# Shared process namespace example
spec:
  shareProcessNamespace: true    # containers see each other's processes
  containers:
    - name: nginx
      image: nginx:1.25
    - name: reloader
      image: busybox
      command:
        - sh
        - -c
        - |
          # Can now send signals to nginx process
          while true; do
            sleep 60
            kill -HUP $(pgrep nginx)    # reload nginx config
          done
```

---

## 🔷 PART 3 — Sidecar in K8s 1.29+ (Native Sidecar)

### The Old Problem

```
Old way (regular containers as sidecars):
  All containers start simultaneously
       ↓
  Main app might start before sidecar is ready
  (e.g. app starts before Envoy proxy is up)
  → App traffic flows without proxy briefly ❌

  Pod shuts down:
  All containers stopped simultaneously
       ↓
  Log shipper killed while still shipping logs
  → Logs lost ❌
```

### Native Sidecar (K8s 1.29+)

```yaml
# native-sidecar.yaml (K8s 1.29+)
apiVersion: v1
kind: Pod
metadata:
  name: app-native-sidecar
spec:
  initContainers:
    # Native sidecar — declared in initContainers
    # but runs for pod lifetime like a regular container
    - name: log-shipper
      image: fluent/fluent-bit:latest
      restartPolicy: Always          # ← this makes it a native sidecar!
                                     # runs alongside main container
                                     # starts BEFORE main container
                                     # stops AFTER main container
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app

  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app

  volumes:
    - name: log-vol
      emptyDir: {}
```

```
Native sidecar lifecycle:
  1. Sidecar starts FIRST (before main container)
  2. Main container starts after sidecar is ready
  3. Main container finishes/dies
  4. Sidecar finishes gracefully AFTER main
  5. No more lost logs on shutdown ✅
```

---

## 🔷 PART 4 — Complete Real World Example

### Django App with Full Sidecar + Init Setup

```yaml
# production-django.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      volumes:
        - name: app-logs
          emptyDir: {}
        - name: config-vol
          emptyDir: {}

      # ── INIT CONTAINERS ─────────────────────────
      initContainers:

        # 1. Wait for DB
        - name: wait-for-db
          image: busybox:1.35
          command:
            - sh
            - -c
            - |
              until nc -z mysql-service 3306; do
                echo "Waiting for MySQL..."
                sleep 2
              done
              echo "MySQL ready ✅"

        # 2. Wait for Redis
        - name: wait-for-redis
          image: busybox:1.35
          command:
            - sh
            - -c
            - |
              until nc -z redis-service 6379; do
                echo "Waiting for Redis..."
                sleep 2
              done
              echo "Redis ready ✅"

        # 3. Run DB migrations
        - name: migrate
          image: myorg/django:v2.1
          command:
            - python
            - manage.py
            - migrate
            - --noinput
          envFrom:
            - configMapRef:
                name: django-config
            - secretRef:
                name: django-secret

        # 4. Collect static files
        - name: collectstatic
          image: myorg/django:v2.1
          command:
            - python
            - manage.py
            - collectstatic
            - --noinput
          volumeMounts:
            - name: config-vol
              mountPath: /static

      # ── MAIN + SIDECAR CONTAINERS ───────────────
      containers:

        # Main: Django application
        - name: django
          image: myorg/django:v2.1
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: django-config
            - secretRef:
                name: django-secret
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/django
            - name: config-vol
              mountPath: /static
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8000
            periodSeconds: 20
            failureThreshold: 3

        # Sidecar: Log shipper
        - name: log-shipper
          image: fluent/fluent-bit:2.3
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/django
              readOnly: true
          env:
            - name: ES_HOST
              value: "elasticsearch-service"
          resources:
            requests:
              cpu: 50m
              memory: 50Mi
            limits:
              cpu: 100m
              memory: 100Mi
```

---

## 🔷 Init vs Sidecar — Full Comparison

```
┌──────────────────┬────────────────────────┬───────────────────────┐
│                  │   Init Container       │   Sidecar Container   │
├──────────────────┼────────────────────────┼───────────────────────┤
│ When runs        │ Before main containers │ With main containers  │
│ Lifetime         │ Runs to completion     │ Runs forever (pod)    │
│ Order            │ Sequential             │ Parallel              │
│ Must succeed?    │ Yes or pod fails       │ No (restarts on fail) │
│ Can have probes? │ No                     │ Yes                   │
│ Typical use      │ Setup, migrations      │ Logging, proxy, mesh  │
│                  │ Wait for deps          │ Config reload         │
└──────────────────┴────────────────────────┴───────────────────────┘
```

---

## ✅ Chapter 16 Summary

| Concept | Key Point |
|---|---|
| Init Container | Runs before main — must complete — sequential |
| Sidecar Container | Runs alongside main — same lifetime — parallel |
| emptyDir volume | Shared scratch space between containers in pod |
| Init use cases | DB migration, wait for deps, pull secrets, git clone |
| Sidecar use cases | Log shipping, proxy, metrics adapter, config reload |
| shareProcessNamespace | Containers can see each other's processes |
| Native Sidecar (1.29+) | restartPolicy: Always in initContainers — proper lifecycle |
| Communication | Localhost networking or shared volumes |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What is an Init Container and when would you use one?"**
   → An Init Container runs before the main container and must complete successfully. Used for waiting for dependencies, running DB migrations, pulling secrets from Vault, or any setup the app needs before starting.

2. **"What is the Sidecar pattern in Kubernetes?"**
   → A helper container that runs alongside the main container in the same pod, sharing network and volumes. Used for log shipping, service mesh proxies, metrics adapters, and config reloading — keeping concerns separate from the main app.

3. **"How do Init Containers and Sidecar Containers differ?"**
   → Init containers run sequentially before main containers and must exit with code 0. Sidecars run in parallel with the main container for the pod's entire lifetime. Init = setup. Sidecar = ongoing helper.

4. **"How do containers in the same pod communicate?"**
   → Via localhost since they share the same network namespace. Or via shared volumes — one container writes, another reads. With shareProcessNamespace they can also send signals to each other's processes.

5. **"What problem does the native sidecar feature in K8s 1.29 solve?"**
   → Old sidecars start and stop simultaneously with main containers — causing race conditions on startup and log loss on shutdown. Native sidecars (restartPolicy: Always in initContainers) start before main and stop after, guaranteeing proper ordering.

---

## 🔷 Roadmap Progress

```
✅ Chapters 1-16 Complete — Core k8s mastered!

🔜 Chapter 17 — Istio Service Mesh
               (the most advanced topic — traffic management,
               mTLS, observability between microservices)

🔜 Chapter 18 — Projects
               Project 1: 3-tier Chat App on Minikube
               Project 2: .NET/Python on KIND
               Project 3: EKS Mega Project
```

---

Ready for **Chapter 17 → Istio Service Mesh** — how traffic flows between microservices, mutual TLS, circuit breaking, and canary deployments at the network level? Say **"next"** 🚀
