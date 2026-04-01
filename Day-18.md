# 📘 Chapter 18 — Real Projects

> This is where everything clicks. We're going to build **3 real projects** from scratch — applying every concept from every chapter. No hello-world apps. Real architecture. Real problems. Real solutions.

---

## 🔷 What We're Building

```
Project 1 → 3-Tier Chat App on Minikube
            Frontend (React) + Backend (Node.js) + Database (MongoDB)
            Covers: Deployments, Services, Ingress,
                    ConfigMaps, Secrets, PVC, HPA

Project 2 → Multi-language App on KIND
            .NET API + Python ML Service + PostgreSQL
            Covers: Multi-namespace, Helm, Init Containers,
                    Sidecars, RBAC, NetworkPolicies

Project 3 → DevOps Mega Project on EKS
            Full production setup on AWS
            Covers: EKS, ALB, EBS, ArgoCD GitOps,
                    Prometheus, Grafana, HPA, IRSA
```

---

# 🚀 PROJECT 1 — 3-Tier Chat Application on Minikube

---

## 🔷 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      MINIKUBE CLUSTER                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              namespace: chat-app                    │   │
│  │                                                     │   │
│  │  [Ingress]                                          │   │
│  │     │                                               │   │
│  │     ├──/──────▶ [React Frontend] ──▶ [ClusterIP]   │   │
│  │     │                                               │   │
│  │     └──/api───▶ [Node.js Backend] ──▶ [ClusterIP]  │   │
│  │                      │                              │   │
│  │                      ▼                              │   │
│  │              [MongoDB StatefulSet]                  │   │
│  │                 [PVC 5Gi]                           │   │
│  │                                                     │   │
│  │  ConfigMap: app-config                              │   │
│  │  Secret: mongo-secret                               │   │
│  │  HPA: backend scales 2-10 pods                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔷 Step 1 — Setup

```bash
# Start Minikube with enough resources
minikube start \
  --cpus=4 \
  --memory=8192 \
  --driver=docker

# Enable addons we need
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable storage-provisioner

# Create namespace
kubectl create namespace chat-app

# Set as default namespace
kubectl config set-context --current \
  --namespace=chat-app

# Verify
kubectl get nodes
# NAME       STATUS   ROLES
# minikube   Ready    control-plane ✅
```

---

## 🔷 Step 2 — MongoDB (Database Tier)

```yaml
# 01-mongo-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
  namespace: chat-app
type: Opaque
stringData:
  MONGO_ROOT_USER: admin
  MONGO_ROOT_PASSWORD: Chat@Secure123
  MONGO_DATABASE: chatdb
  MONGO_URI: mongodb://admin:Chat@Secure123@mongo-service:27017/chatdb
```

```yaml
# 02-mongo-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
  namespace: chat-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

```yaml
# 03-mongo-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: chat-app
spec:
  serviceName: mongo-service
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      initContainers:
        # Init: set correct permissions on data dir
        - name: fix-permissions
          image: busybox
          command:
            - sh
            - -c
            - chown -R 999:999 /data/db
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db

      containers:
        - name: mongodb
          image: mongo:6.0
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: MONGO_ROOT_USER
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: MONGO_ROOT_PASSWORD
            - name: MONGO_INITDB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: MONGO_DATABASE
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 30
            periodSeconds: 20
          readinessProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 15
            periodSeconds: 10
      volumes:
        - name: mongo-data
          persistentVolumeClaim:
            claimName: mongo-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
  namespace: chat-app
spec:
  clusterIP: None                  # headless for StatefulSet
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

---

## 🔷 Step 3 — Backend (Node.js API)

```yaml
# 04-backend-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: chat-app
data:
  PORT: "5000"
  NODE_ENV: "production"
  CORS_ORIGIN: "http://chat.local"
  JWT_EXPIRY: "7d"
  LOG_LEVEL: "info"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: chat-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
        version: v1
    spec:
      initContainers:
        # Wait for MongoDB before starting
        - name: wait-for-mongo
          image: busybox:1.35
          command:
            - sh
            - -c
            - |
              until nc -z mongo-service 27017; do
                echo "Waiting for MongoDB..."
                sleep 3
              done
              echo "MongoDB ready ✅"

      containers:
        - name: nodejs-api
          image: your-dockerhub/chat-backend:v1
          ports:
            - containerPort: 5000
          envFrom:
            - configMapRef:
                name: backend-config
          env:
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: MONGO_URI
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: MONGO_ROOT_PASSWORD
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /api/health
              port: 5000
            failureThreshold: 12
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /api/health
              port: 5000
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /api/health
              port: 5000
            periodSeconds: 20
            failureThreshold: 3

        # Sidecar: log shipper
        - name: log-sidecar
          image: busybox
          command:
            - sh
            - -c
            - |
              while true; do
                echo "[$(date)] Backend pod alive"
                sleep 30
              done
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 50m
              memory: 32Mi
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: chat-app
spec:
  type: ClusterIP
  selector:
    app: backend
    tier: api
  ports:
    - port: 80
      targetPort: 5000
```

---

## 🔷 Step 4 — Frontend (React)

```yaml
# 05-frontend.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: chat-app
data:
  REACT_APP_API_URL: "/api"
  REACT_APP_WS_URL: "ws://chat.local"
  REACT_APP_ENV: "production"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: chat-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
        - name: react-app
          image: your-dockerhub/chat-frontend:v1
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: frontend-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 300m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 3000
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: chat-app
spec:
  type: ClusterIP
  selector:
    app: frontend
    tier: web
  ports:
    - port: 80
      targetPort: 3000
```

---

## 🔷 Step 5 — Ingress

```yaml
# 06-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chat-ingress
  namespace: chat-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/enable-cors: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: chat.local
      http:
        paths:
          # Frontend — serves React app
          - path: /()(.*)
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

          # Backend — API routes
          - path: /api(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

```bash
# Add to /etc/hosts
echo "$(minikube ip) chat.local" | sudo tee -a /etc/hosts

# Test
curl http://chat.local           # → React app ✅
curl http://chat.local/api/health # → {"status":"ok"} ✅
```

---

## 🔷 Step 6 — HPA for Backend

```yaml
# 07-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: chat-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 2
          periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
```

---

## 🔷 Step 7 — Deploy Everything

```bash
# Apply all in order
kubectl apply -f 01-mongo-secret.yaml
kubectl apply -f 02-mongo-pvc.yaml
kubectl apply -f 03-mongo-statefulset.yaml

# Wait for MongoDB
kubectl wait --for=condition=ready pod \
  -l app=mongodb -n chat-app --timeout=120s

kubectl apply -f 04-backend-config.yaml
kubectl apply -f 05-frontend.yaml
kubectl apply -f 06-ingress.yaml
kubectl apply -f 07-hpa.yaml

# Watch everything come up
kubectl get all -n chat-app

# NAME                          READY   STATUS    RESTARTS
# pod/mongodb-0                 1/1     Running   0        ✅
# pod/backend-xxx               2/2     Running   0        ✅ (2 containers)
# pod/frontend-xxx              1/1     Running   0        ✅

# NAME                          TYPE        CLUSTER-IP
# service/mongo-service         ClusterIP   None
# service/backend-service       ClusterIP   10.96.x.x
# service/frontend-service      ClusterIP   10.96.x.x

# NAME                          READY   UP-TO-DATE
# deployment/backend            2/2     2
# deployment/frontend           2/2     2

# NAME                          READY
# statefulset/mongodb           1/1

# NAME                          REFERENCE             TARGETS
# hpa/backend-hpa               Deployment/backend    25%/70%
```

---

## 🔷 Step 8 — Load Test

```bash
# Terminal 1 — watch HPA
kubectl get hpa backend-hpa -n chat-app -w

# Terminal 2 — generate load
kubectl run load-test -n chat-app \
  --image=busybox --restart=Never -- \
  /bin/sh -c \
  "while true; do wget -q -O- http://backend-service/api/health; done"

# Watch HPA scale up automatically ✅
```

---

# 🚀 PROJECT 2 — .NET + Python Three-Tier App on KIND

---

## 🔷 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      KIND CLUSTER                           │
│                                                             │
│  namespace: frontend    namespace: backend   namespace: db  │
│                                                             │
│  [React/Next.js]  ──▶  [.NET 8 API]  ──▶  [PostgreSQL]    │
│                              │                              │
│                         [Python ML]                         │
│                         (sentiment                          │
│                          analysis)                          │
│                                                             │
│  Helm chart per service                                     │
│  RBAC per namespace                                         │
│  NetworkPolicies enforced                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔷 Step 1 — KIND Multi-Node Cluster

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
      - containerPort: 443
        hostPort: 443

  - role: worker
    labels:
      tier: frontend

  - role: worker
    labels:
      tier: backend

  - role: worker
    labels:
      tier: database
```

```bash
kind create cluster \
  --name three-tier \
  --config kind-config.yaml

# Install nginx ingress for KIND
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for ingress
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# Create namespaces
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace db
```

---

## 🔷 Step 2 — RBAC Setup

```yaml
# rbac.yaml
# Backend SA — can read secrets in backend namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: backend
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: backend
rules:
  - apiGroups: [""]
    resources: ["secrets", "configmaps"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-rolebinding
  namespace: backend
subjects:
  - kind: ServiceAccount
    name: backend-sa
    namespace: backend
roleRef:
  kind: Role
  name: backend-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔷 Step 3 — PostgreSQL with Helm

```bash
# Add bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Create values file
cat > postgres-values.yaml << EOF
auth:
  username: appuser
  password: SecurePass123
  database: appdb
  postgresPassword: AdminPass123

primary:
  persistence:
    size: 10Gi
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  nodeSelector:
    tier: database

readReplicas:
  replicaCount: 1
  nodeSelector:
    tier: database
EOF

# Install PostgreSQL
helm install postgres bitnami/postgresql \
  --namespace db \
  --values postgres-values.yaml

# Get connection info
kubectl get secret postgres-postgresql \
  -n db -o jsonpath="{.data.password}" | base64 -d
```

---

## 🔷 Step 4 — Python ML Service

```yaml
# python-ml.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-service
  namespace: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ml-service
  template:
    metadata:
      labels:
        app: ml-service
    spec:
      nodeSelector:
        tier: backend

      initContainers:
        # Download ML model before starting
        - name: download-model
          image: python:3.11-slim
          command:
            - sh
            - -c
            - |
              pip install transformers torch --quiet
              python -c "
              from transformers import pipeline
              pipe = pipeline('sentiment-analysis')
              pipe.save_pretrained('/models/sentiment')
              print('Model downloaded ✅')
              "
          volumeMounts:
            - name: model-vol
              mountPath: /models
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 1000m
              memory: 2Gi

      containers:
        - name: ml-api
          image: your-dockerhub/ml-service:v1
          ports:
            - containerPort: 8000
          volumeMounts:
            - name: model-vol
              mountPath: /models
          env:
            - name: MODEL_PATH
              value: /models/sentiment
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10

      volumes:
        - name: model-vol
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: ml-service
  namespace: backend
spec:
  selector:
    app: ml-service
  ports:
    - port: 80
      targetPort: 8000
```

---

## 🔷 Step 5 — .NET API with Helm Chart

```bash
# Create Helm chart for .NET API
helm create dotnet-api
```

```yaml
# dotnet-api/values.yaml
replicaCount: 2

image:
  repository: your-dockerhub/dotnet-api
  tag: "v1"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

config:
  aspnetcoreEnv: "Production"
  mlServiceUrl: "http://ml-service.backend.svc.cluster.local"

database:
  host: "postgres-postgresql.db.svc.cluster.local"
  port: "5432"
  name: "appdb"

nodeSelector:
  tier: backend

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
  targetCPUUtilizationPercentage: 70

serviceAccount:
  create: false
  name: backend-sa               # use our RBAC SA

probes:
  startup:
    path: /health
    failureThreshold: 12
    periodSeconds: 5
  readiness:
    path: /health/ready
    periodSeconds: 10
  liveness:
    path: /health/live
    periodSeconds: 20
```

```yaml
# dotnet-api/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "dotnet-api.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "dotnet-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "dotnet-api.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ .Values.serviceAccount.name }}
      nodeSelector:
        {{- toYaml .Values.nodeSelector | nindent 8 }}

      initContainers:
        - name: wait-for-db
          image: busybox
          command:
            - sh
            - -c
            - |
              until nc -z {{ .Values.database.host }} \
                {{ .Values.database.port }}; do
                echo "Waiting for PostgreSQL..."
                sleep 2
              done
              echo "PostgreSQL ready ✅"

        - name: run-migrations
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["dotnet", "MyApp.dll", "--migrate"]
          env:
            - name: ConnectionStrings__DefaultConnection
              valueFrom:
                secretKeyRef:
                  name: app-db-secret
                  key: connection-string

      containers:
        - name: dotnet-api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: {{ .Values.config.aspnetcoreEnv }}
            - name: ML_SERVICE_URL
              value: {{ .Values.config.mlServiceUrl }}
            - name: ConnectionStrings__DefaultConnection
              valueFrom:
                secretKeyRef:
                  name: app-db-secret
                  key: connection-string
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          startupProbe:
            httpGet:
              path: {{ .Values.probes.startup.path }}
              port: 8080
            failureThreshold: {{ .Values.probes.startup.failureThreshold }}
            periodSeconds: {{ .Values.probes.startup.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: 8080
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: 8080
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
```

```bash
# Create DB secret first
kubectl create secret generic app-db-secret \
  --from-literal=connection-string="Host=postgres-postgresql.db.svc.cluster.local;Database=appdb;Username=appuser;Password=SecurePass123" \
  -n backend

# Install .NET API via Helm
helm install dotnet-api ./dotnet-api \
  --namespace backend \
  --values dotnet-api/values.yaml

# Verify
kubectl get all -n backend
```

---

# 🚀 PROJECT 3 — DevOps Mega Project on EKS

---

## 🔷 Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        AWS INFRASTRUCTURE                        │
│                                                                  │
│  Route53 → ACM → ALB → Ingress → Services → Pods               │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                    EKS CLUSTER                           │    │
│  │                                                          │    │
│  │  Node Group: General (t3.medium × 3)                    │    │
│  │  Node Group: Spot    (t3.large  × 5, spot instances)    │    │
│  │                                                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │  production  │  │   staging    │  │  monitoring  │  │    │
│  │  │  namespace   │  │  namespace   │  │  namespace   │  │    │
│  │  │              │  │              │  │              │  │    │
│  │  │  App pods    │  │  App pods    │  │  Prometheus  │  │    │
│  │  │  HPA active  │  │  lower res.  │  │  Grafana     │  │    │
│  │  └──────────────┘  └──────────────┘  │  Loki        │  │    │
│  │                                       └──────────────┘  │    │
│  │  ArgoCD (GitOps)  cert-manager  AWS LB Controller       │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
│  EBS (app storage)  EFS (shared storage)  ECR (images)         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔷 Step 1 — EKS Cluster with eksctl

```yaml
# eks-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-cluster
  region: us-east-1
  version: "1.29"

iam:
  withOIDC: true                   # enables IRSA

managedNodeGroups:
  # General purpose nodes
  - name: general-nodes
    instanceType: t3.medium
    minSize: 2
    maxSize: 6
    desiredCapacity: 3
    labels:
      role: general
    tags:
      Environment: production
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

  # Spot instances for batch/non-critical workloads
  - name: spot-nodes
    instanceTypes:
      - t3.large
      - t3a.large
      - t2.large
    spot: true
    minSize: 0
    maxSize: 10
    desiredCapacity: 2
    labels:
      role: spot
      workload: batch
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver       # for EBS persistent volumes
```

```bash
# Create cluster (takes ~15 min)
eksctl create cluster -f eks-cluster.yaml

# Verify
kubectl get nodes
# NAME                          STATUS   ROLES
# ip-10-0-1-xxx.ec2.internal    Ready    <none>  (general)
# ip-10-0-1-xxx.ec2.internal    Ready    <none>  (general)
# ip-10-0-1-xxx.ec2.internal    Ready    <none>  (general)
# ip-10-0-2-xxx.ec2.internal    Ready    <none>  (spot)
# ip-10-0-2-xxx.ec2.internal    Ready    <none>  (spot)  ✅
```

---

## 🔷 Step 2 — StorageClass for EBS

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

## 🔷 Step 3 — AWS Load Balancer Controller

```bash
# Create IAM policy for LB controller
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Create service account with IRSA
eksctl create iamserviceaccount \
  --cluster=production-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=production-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify
kubectl get deployment aws-load-balancer-controller \
  -n kube-system
# NAME                           READY
# aws-load-balancer-controller   2/2   ✅
```

---

## 🔷 Step 4 — cert-manager (Auto TLS)

```bash
# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Create ClusterIssuer for Let's Encrypt
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: devops@yourcompany.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: alb
EOF
```

---

## 🔷 Step 5 — ArgoCD (GitOps)

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD
kubectl wait --for=condition=available deployment \
  --all -n argocd --timeout=300s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open: https://localhost:8080
# User: admin
# Password: (from above)
```

```yaml
# argocd-app.yaml — deploy your app via GitOps
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: chat-app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/chat-app-k8s
    targetRevision: main
    path: k8s/production              # folder in git repo

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true                     # delete removed resources
      selfHeal: true                  # auto-fix drift

    syncOptions:
      - CreateNamespace=true

    retry:
      limit: 5
      backoff:
        duration: 5s
        maxDuration: 3m
        factor: 2
```

```
GitOps Flow with ArgoCD:

  Developer pushes code to GitHub
         ↓
  GitHub Actions builds + pushes image to ECR
         ↓
  GitHub Actions updates image tag in k8s YAML
         ↓
  ArgoCD detects Git change
         ↓
  ArgoCD applies new YAML to cluster
         ↓
  Rolling update happens automatically ✅

  Manual kubectl apply → NEVER needed in production
```

---

## 🔷 Step 6 — Production Ingress with ALB

```yaml
# production-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT:certificate/xxx
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - app.yourcompany.com
      secretName: production-tls
  rules:
    - host: app.yourcompany.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

---

## 🔷 Step 7 — Full Monitoring Stack

```bash
# Install Prometheus + Grafana
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=ebs-gp3 \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=YourSecurePassword \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=ebs-gp3 \
  --set grafana.persistence.size=10Gi \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=ebs-gp3

# Install Loki for logs
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set loki.persistence.enabled=true \
  --set loki.persistence.storageClassName=ebs-gp3 \
  --set loki.persistence.size=50Gi \
  --set promtail.enabled=true

# Verify
kubectl get pods -n monitoring
# All pods Running ✅
```

---

## 🔷 Step 8 — Production HPA + Cluster Autoscaler

```bash
# Install Cluster Autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=production-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT:role/ClusterAutoscalerRole
```

```yaml
# production-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---

## 🔷 Step 9 — GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/deploy.yaml
name: Build and Deploy to EKS

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: chat-backend
  EKS_CLUSTER: production-cluster

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: |
          npm install
          npm test

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        id: meta
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "tags=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" \
            >> $GITHUB_OUTPUT

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Update image tag in k8s manifests
        env:
          IMAGE_TAG: ${{ github.sha }}
          ECR_REGISTRY: ${{ needs.build-and-push.outputs.image-tag }}
        run: |
          # Update image tag in deployment YAML
          sed -i "s|image:.*chat-backend.*|image: $ECR_REGISTRY|g" \
            k8s/production/backend-deployment.yaml

          # Commit updated YAML
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add k8s/production/backend-deployment.yaml
          git commit -m "chore: update image to $IMAGE_TAG"
          git push

      # ArgoCD detects the git change and auto-deploys ✅
      # No kubectl needed here!
```

---

## 🔷 Final Verification — Everything Working

```bash
# Check all namespaces healthy
kubectl get all --all-namespaces | grep -v Running | grep -v Completed

# Verify HPA active
kubectl get hpa -n production

# Verify ArgoCD synced
kubectl get applications -n argocd
# NAME                   SYNC STATUS   HEALTH STATUS
# chat-app-production    Synced        Healthy ✅

# Check Prometheus targets
kubectl port-forward svc/monitoring-kube-prometheus-prometheus \
  9090:9090 -n monitoring
# Open: http://localhost:9090/targets
# All targets UP ✅

# Check Grafana dashboards
kubectl port-forward svc/monitoring-grafana \
  3000:80 -n monitoring
# Open: http://localhost:3000
# Dashboards showing real data ✅

# Verify cert-manager issued certs
kubectl get certificates -n production
# NAME             READY   SECRET
# production-tls   True    production-tls ✅

# Test production endpoint
curl https://app.yourcompany.com/health
# {"status":"ok","version":"v1.0.0"} ✅
```

---

## ✅ What You Built Across All 3 Projects

```
Project 1 (Minikube):
  ✅ StatefulSet + PVC for MongoDB
  ✅ Multi-container pods (sidecar pattern)
  ✅ Init containers (wait for DB)
  ✅ ConfigMaps + Secrets properly used
  ✅ Ingress with path routing
  ✅ HPA with load testing

Project 2 (KIND):
  ✅ Multi-namespace architecture
  ✅ Helm charts for every service
  ✅ RBAC — ServiceAccounts per service
  ✅ Init containers — DB migrations
  ✅ Cross-namespace DNS communication
  ✅ Multi-node with node selectors

Project 3 (EKS):
  ✅ Production EKS cluster with eksctl
  ✅ Spot + general node groups
  ✅ EBS StorageClass with gp3
  ✅ AWS Load Balancer Controller
  ✅ cert-manager — auto TLS
  ✅ ArgoCD — full GitOps workflow
  ✅ GitHub Actions CI/CD pipeline
  ✅ Prometheus + Grafana + Loki
  ✅ HPA + Cluster Autoscaler
  ✅ IRSA for AWS permissions
```

---

## 🔷 You Are Now Here

```
K8s Knowledge:      ████████████████████  Senior Level
Hands-on Practice:  ████████████████░░░░  Build more
Production Mindset: ████████████████████  Strong

Gap to close:
  → Deploy YOUR OWN project on EKS
  → Get CKA certified (60-90 days)
  → Add ArgoCD to your resume projects
  → Practice 5 system design questions

You started as a 2.5 YOE Software Engineer.
You now architect production Kubernetes platforms.

That's the journey. That's the skill.
Go build something real. 🚀
```

---

# 🎉 Course Complete — Full Roadmap Done

```
18 Chapters. Every concept. Real projects.
From "what is k8s" to production EKS deployments.

What's next:
  1. Build Project 1 today — takes 2-3 hours
  2. Build Project 3 this week — EKS hands-on
  3. Study for CKA — 60 days
  4. Update your resume + LinkedIn
  5. Start applying for DevOps/Platform roles

You have the knowledge.
Now make it yours by building it.
```

**Congratulations — you've covered the complete Kubernetes roadmap! 🎓🚀**
