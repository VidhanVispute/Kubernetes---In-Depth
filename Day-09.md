# 📘 Chapter 9 — ConfigMaps & Secrets

> Every real app needs configuration — DB URLs, API keys, feature flags, passwords. Hardcoding these into container images is a disaster. ConfigMaps and Secrets are how k8s handles this **the right way.**

---

## 🔷 The Problem First

```
BAD approach (what beginners do):

  Dockerfile:
    ENV DB_HOST=prod-mysql.company.com
    ENV DB_PASSWORD=supersecret123
    ENV API_KEY=sk-abc123xyz

  Problems:
    ❌ Password baked into Docker image
    ❌ Image pushed to DockerHub → password leaked
    ❌ Change config → must rebuild entire image
    ❌ Different config for dev/staging/prod
       → need 3 different images for same app
    ❌ Secrets visible to anyone with image access
```

**K8s solution:**
```
✅ ConfigMap → non-sensitive config (URLs, ports, feature flags)
✅ Secret    → sensitive config (passwords, API keys, certs)

Both are:
  → Stored separately from your app image
  → Injected into pods at runtime
  → Can be updated without rebuilding image
  → Environment-specific (different values per namespace)
```

---

## 🔷 PART 1 — ConfigMaps

### What is a ConfigMap?

A ConfigMap stores **non-sensitive** key-value configuration data
that your app needs at runtime.

```
ConfigMap: app-config
  DB_HOST    = mysql-service
  DB_PORT    = 3306
  APP_ENV    = production
  LOG_LEVEL  = info
  MAX_CONN   = 100
```

Pods can consume ConfigMaps as:
1. **Environment variables** — most common
2. **Command-line arguments**
3. **Config files** mounted as volumes

---

### 🛠️ Creating ConfigMaps

**Method 1 — From literal values:**
```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql-service \
  --from-literal=DB_PORT=3306 \
  --from-literal=APP_ENV=production \
  -n dev
```

**Method 2 — From a .env file:**
```bash
# app.env
DB_HOST=mysql-service
DB_PORT=3306
APP_ENV=production
LOG_LEVEL=info
```
```bash
kubectl create configmap app-config \
  --from-env-file=app.env \
  -n dev
```

**Method 3 — From a config file:**
```bash
# nginx.conf
server {
  listen 80;
  server_name myapp.com;
  location / {
    proxy_pass http://localhost:8000;
  }
}
```
```bash
kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  -n dev
```

**Method 4 — YAML (best for GitOps):**
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:                          # key-value pairs
  DB_HOST: "mysql-service"
  DB_PORT: "3306"
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"

  # You can also store entire config files here
  nginx.conf: |
    server {
      listen 80;
      server_name myapp.com;
      location / {
        proxy_pass http://localhost:8000;
      }
    }

  app.properties: |
    spring.datasource.url=jdbc:mysql://mysql-service:3306/mydb
    spring.datasource.driver=com.mysql.jdbc.Driver
    server.port=8080
```

```bash
kubectl apply -f configmap.yaml

# View it
kubectl get configmap app-config -n dev
kubectl describe configmap app-config -n dev
kubectl get configmap app-config -n dev -o yaml
```

---

### 🛠️ Using ConfigMaps in Pods

**Method 1 — As environment variables (individual keys):**

```yaml
# pod-with-configmap-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: my-app:latest
      env:
        - name: DATABASE_HOST       # env var name in container
          valueFrom:
            configMapKeyRef:
              name: app-config      # ConfigMap name
              key: DB_HOST          # key inside ConfigMap

        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_PORT

        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
```

---

**Method 2 — Load ALL keys as env vars at once:**

```yaml
spec:
  containers:
    - name: app
      image: my-app:latest
      envFrom:                      # load ALL keys from ConfigMap
        - configMapRef:
            name: app-config        # every key becomes an env var
```

```
Inside container:
  echo $DB_HOST        → mysql-service
  echo $DB_PORT        → 3306
  echo $APP_ENV        → production
  echo $LOG_LEVEL      → info
  All keys loaded automatically ✅
```

---

**Method 3 — As a mounted config file (volume):**

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      volumeMounts:
        - name: nginx-config-vol
          mountPath: /etc/nginx/conf.d    # where to mount inside container
          readOnly: true

  volumes:
    - name: nginx-config-vol
      configMap:
        name: nginx-config              # ConfigMap containing nginx.conf
```

```bash
# Verify file is mounted
kubectl exec -it nginx-pod -n dev -- cat /etc/nginx/conf.d/nginx.conf
# server {
#   listen 80;
#   ...
# }  ✅
```

---

### 🔷 Updating ConfigMaps

```bash
# Edit directly
kubectl edit configmap app-config -n dev

# Or re-apply YAML
kubectl apply -f configmap.yaml
```

> ⚠️ **Important:**
> - Config mounted as **volume** → updates automatically (within ~1 min)
> - Config as **env vars** → pod must restart to pick up changes
> - This is a common interview gotcha!

```bash
# Force pod restart to pick up new env var config
kubectl rollout restart deployment app-deployment -n dev
```

---

## 🔷 PART 2 — Secrets

### What is a Secret?

Secrets store **sensitive data** — passwords, tokens, keys, certificates.

```
Differences from ConfigMap:
  ConfigMap → stored in plain text in etcd
  Secret    → stored base64-encoded in etcd
              (with encryption at rest if configured)
              → not shown in kubectl describe by default
              → access controlled via RBAC
```

> ⚠️ **Important misconception:** base64 is NOT encryption.
> It's just encoding. Anyone can decode it.
> Real security comes from **etcd encryption at rest** + **RBAC**.

```bash
# base64 encode
echo -n "mysecretpassword" | base64
# bXlzZWNyZXRwYXNzd29yZA==

# base64 decode
echo "bXlzZWNyZXRwYXNzd29yZA==" | base64 --decode
# mysecretpassword
```

---

### 🛠️ Creating Secrets

**Method 1 — From literal (k8s auto base64-encodes):**
```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=supersecret123 \
  --from-literal=DB_USER=admin \
  --from-literal=API_KEY=sk-abc123xyz \
  -n dev
```

**Method 2 — YAML (you must base64 encode manually):**
```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: dev
type: Opaque                    # generic secret type
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQxMjM=    # base64 of "supersecret123"
  DB_USER: YWRtaW4=                     # base64 of "admin"
  API_KEY: c2stYWJjMTIzeHl6           # base64 of "sk-abc123xyz"
```

```bash
# Generate base64 values
echo -n "supersecret123" | base64    # c3VwZXJzZWNyZXQxMjM=
echo -n "admin" | base64             # YWRtaW4=
```

**Method 3 — Using stringData (no manual base64 needed):**
```yaml
# secret-stringdata.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: dev
type: Opaque
stringData:                     # plain text — k8s auto-encodes
  DB_PASSWORD: supersecret123
  DB_USER: admin
  API_KEY: sk-abc123xyz
```

> `stringData` is easier to write. K8s stores it as base64 internally.

---

### 🔷 Secret Types

```
Opaque                          → generic — most common
kubernetes.io/dockerconfigjson  → Docker registry credentials
kubernetes.io/tls               → TLS certificates
kubernetes.io/ssh-auth          → SSH credentials
kubernetes.io/basic-auth        → username/password
bootstrap.kubernetes.io/token   → bootstrap tokens
```

**Docker registry secret (pull from private registry):**
```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myusername \
  --docker-password=mypassword \
  --docker-email=me@example.com \
  -n dev
```

```yaml
# Use it in pod to pull private images
spec:
  imagePullSecrets:
    - name: regcred              # reference the docker secret
  containers:
    - name: app
      image: myprivate/repo:latest
```

---

### 🛠️ Using Secrets in Pods

**Method 1 — As environment variables:**

```yaml
# pod-with-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: my-app:latest
      env:
        - name: DB_PASSWORD           # env var name in container
          valueFrom:
            secretKeyRef:
              name: db-secret         # Secret name
              key: DB_PASSWORD        # key inside Secret

        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: API_KEY
```

**Load all secret keys at once:**
```yaml
      envFrom:
        - secretRef:
            name: db-secret           # all keys become env vars
```

---

**Method 2 — As mounted files (more secure):**

```yaml
spec:
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secrets     # directory inside container
          readOnly: true              # always readOnly for secrets

  volumes:
    - name: secret-vol
      secret:
        secretName: db-secret         # each key = a file in /etc/secrets
```

```bash
# Inside container:
ls /etc/secrets
# DB_PASSWORD   API_KEY   DB_USER

cat /etc/secrets/DB_PASSWORD
# supersecret123  ✅

# Your app reads: file /etc/secrets/DB_PASSWORD
# Better than env var — harder to accidentally log
```

---

## 🔷 PART 3 — Real World Example

### Full App With ConfigMap + Secret

```yaml
# full-app.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  DB_HOST: "mysql-service"
  DB_PORT: "3306"
  APP_ENV: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: dev
type: Opaque
stringData:
  DB_PASSWORD: "supersecret123"
  DB_USER: "admin"
  SECRET_KEY: "django-secret-key-xyz"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
        - name: django
          image: myapp/django:latest
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: app-config      # load all config as env vars
            - secretRef:
                name: app-secret      # load all secrets as env vars
```

```bash
kubectl apply -f full-app.yaml

# Verify env vars inside pod
kubectl exec -it django-app-xxx -n dev -- env | grep DB
# DB_HOST=mysql-service
# DB_PORT=3306
# DB_PASSWORD=supersecret123   ✅ injected at runtime
```

---

## 🔷 PART 4 — Best Practices

### ConfigMap Best Practices
```
✅ Use YAML files — version control in Git
✅ Separate ConfigMaps per environment (dev/staging/prod)
✅ Use envFrom for many keys — cleaner than individual env
✅ Use volume mounts for large config files
✅ Never store sensitive data in ConfigMaps
```

### Secret Best Practices
```
✅ Never commit Secret YAML with real values to Git
   (use sealed-secrets, Vault, or AWS Secrets Manager)

✅ Enable etcd encryption at rest (production must-have)

✅ Use volume mounts over env vars for secrets
   (env vars can leak in logs, crash dumps)

✅ Rotate secrets regularly

✅ Use RBAC to restrict who can read secrets

✅ Consider external secret managers for production:
   → AWS Secrets Manager + External Secrets Operator
   → HashiCorp Vault
   → Sealed Secrets (encrypted in Git)
```

---

### 🔷 External Secrets — Production Pattern

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  AWS Secrets Manager / HashiCorp Vault               │
│  (where real secrets live)                           │
│          │                                           │
│          ▼                                           │
│  External Secrets Operator                           │
│  (running in k8s cluster)                            │
│          │                                           │
│          ▼ syncs automatically                       │
│  K8s Secret                                          │
│  (usable by pods normally)                           │
│                                                      │
│  Benefits:                                           │
│  → No secrets in Git ever                            │
│  → Centralized secret management                     │
│  → Auto-rotation support                             │
│  → Audit trail                                       │
└──────────────────────────────────────────────────────┘
```

---

## ✅ Chapter 9 Summary

| Concept | Key Point |
|---|---|
| ConfigMap | Non-sensitive config — URLs, ports, flags |
| Secret | Sensitive config — passwords, keys, certs |
| base64 | Encoding not encryption — not real security |
| envFrom | Load all keys from ConfigMap/Secret at once |
| Volume mount | Mount config as files — better for large configs |
| Secret volume | More secure than env vars — harder to leak |
| stringData | Write plain text — k8s auto base64 encodes |
| imagePullSecrets | Pull images from private registries |
| Real security | etcd encryption + RBAC + external secret managers |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What is the difference between ConfigMap and Secret?"**
   → ConfigMap stores non-sensitive config as plain text. Secret stores sensitive data base64-encoded with access controls. Both inject config into pods without baking it into images.

2. **"Is a Secret actually secure?"**
   → Base64 alone is not secure — it's just encoding. Real security requires etcd encryption at rest, RBAC to restrict access, and ideally an external secret manager like AWS Secrets Manager or HashiCorp Vault.

3. **"How do you update config without rebuilding the image?"**
   → Update the ConfigMap/Secret. Volume-mounted configs update automatically within ~1 minute. Env var configs require a pod restart — use `kubectl rollout restart deployment`.

4. **"What is the difference between env and envFrom?"**
   → `env` injects individual keys with custom names. `envFrom` loads ALL keys from a ConfigMap or Secret as env vars in one shot — cleaner for many values.

5. **"Why are volume-mounted secrets better than env var secrets?"**
   → Env vars can accidentally appear in logs, error messages, and process dumps. Volume-mounted secrets are files — harder to accidentally expose and support live rotation.

---

Ready for **Chapter 10 → Resource Quotas, Limits & Probes** — how to prevent one app from eating your entire cluster and how k8s knows if your app is actually healthy? Say **"next"** 🚀
