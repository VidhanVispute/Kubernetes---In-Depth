# 📘 Chapter 6 — Jobs, CronJobs & StatefulSets

> Three different but critical workload types. Jobs for batch tasks, CronJobs for scheduled tasks, StatefulSets for databases and anything that needs **identity + persistent storage.**

---

## 🔷 PART 1 — Jobs

### What is a Job?

Every resource we've seen so far runs **forever:**
```
Deployment  → keep specified number of pods running, supports rolling updates
DaemonSet   → run one pod on every node
ReplicaSet  → maintain a fixed number of identical pods
Job         → run a pod until the task finishes successfully
CronJob     → run jobs on a schedule (like cron)
StatefulSet → manage stateful pods with stable identity and storage
```

But sometimes you need a pod that:
- Runs a task
- **Finishes**
- **Dies** ✅

```
Examples:
  → Database migration script
  → Send batch emails
  → Generate a report
  → Process a file
  → One-time data transformation
```

**That's a Job.**

---

### 🔷 Job vs Deployment

```
┌─────────────────────────────────────────────────┐
│  DEPLOYMENT                                     │
│  "Run nginx, if it dies → restart it forever"   │
│                                                 │
│  Pod dies → ReplicaSet restarts it  🔄          │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  JOB                                            │
│  "Run this script, when done → stop"            │
│                                                 │
│  Pod completes → Job marks it done  ✅          │
│  Pod fails     → Job retries it     🔁          │
└─────────────────────────────────────────────────┘
```

---

### 🛠️ Your First Job

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
  namespace: dev
spec:
  completions: 1          # how many pods must complete successfully
  parallelism: 1          # how many pods run at the same time
  backoffLimit: 4         # retry up to 4 times before marking as failed
  template:
    spec:
      restartPolicy: Never        # Never or OnFailure — NOT Always
      containers:
        - name: batch-container
          image: busybox
          command:
            - sh
            - -c
            - |
              echo "Starting batch job..."
              sleep 10
              echo "Batch job complete!"
```

```bash
# Apply it
kubectl apply -f job.yaml

# Watch it run
kubectl get jobs -n dev
# NAME        COMPLETIONS   DURATION   AGE
# batch-job   0/1           5s         5s
# batch-job   1/1           12s        12s   ✅ done!

# See the pod it created
kubectl get pods -n dev
# NAME              READY   STATUS      RESTARTS
# batch-job-x7k2p   0/1     Completed   0        ✅

# Logs
kubectl logs batch-job-x7k2p -n dev
# Starting batch job...
# Batch job complete!
```

---

### 🔷 Parallel Jobs — Run Multiple Pods at Once

```yaml
# parallel-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 6       # need 6 successful completions total
  parallelism: 2       # run 2 pods at a time
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox
          command: ['sh', '-c', 'echo Processing item && sleep 5']
```

```
Timeline:
  t=0s:  Pod-1 starts, Pod-2 starts     (2 parallel)
  t=5s:  Pod-1 done, Pod-2 done
         Pod-3 starts, Pod-4 starts     (2 parallel)
  t=10s: Pod-3 done, Pod-4 done
         Pod-5 starts, Pod-6 starts     (2 parallel)
  t=15s: All 6 complete ✅
```

---

### 🔷 Job Failure & Retry

```yaml
spec:
  backoffLimit: 4       # retry 4 times
  template:
    spec:
      restartPolicy: OnFailure   # restart container in same pod
      # OR
      restartPolicy: Never       # create a new pod on failure
```

```
backoffLimit: 4 with restartPolicy: Never

Attempt 1 → Pod-1 fails → new Pod-2 created
Attempt 2 → Pod-2 fails → new Pod-3 created
Attempt 3 → Pod-3 fails → new Pod-4 created
Attempt 4 → Pod-4 fails → new Pod-5 created
Attempt 5 → Pod-5 fails → Job marked FAILED ❌
```

---

### 🛠️ Job Commands

```bash
# List jobs
kubectl get jobs -n dev

# Describe job (see events, status)
kubectl describe job batch-job -n dev

# Delete job (also deletes its pods)
kubectl delete job batch-job -n dev

# Get pods created by a job
kubectl get pods -l job-name=batch-job -n dev
```

---

## 🔷 PART 2 — CronJobs

### What is a CronJob?

A CronJob = **Job + Schedule**

```
"Run this Job every day at midnight"
"Run this Job every Monday at 9 AM"
"Run this Job every 5 minutes"
```

Real use cases:
- Database backups every night
- Send weekly report emails
- Clean up old log files every hour
- Sync data every 30 minutes
- Health check pings every 5 minutes

---

### 🔷 Cron Syntax (Quick Refresher)

```
 ┌──────── minute        (0-59)
 │ ┌────── hour          (0-23)
 │ │ ┌──── day of month  (1-31)
 │ │ │ ┌── month         (1-12)
 │ │ │ │ ┌ day of week   (0-6, 0=Sunday)
 │ │ │ │ │
 * * * * *

Examples:
  0 0 * * *     → every day at midnight
  0 9 * * 1     → every Monday at 9 AM
  */5 * * * *   → every 5 minutes
  0 */6 * * *   → every 6 hours
  0 0 1 * *     → first day of every month
```

---

### 🛠️ CronJob YAML

```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: dev
spec:
  schedule: "0 0 * * *"           # every day at midnight
  concurrencyPolicy: Forbid        # don't run if previous still running
  successfulJobsHistoryLimit: 3    # keep last 3 successful jobs
  failedJobsHistoryLimit: 1        # keep last 1 failed job
  startingDeadlineSeconds: 60      # if missed schedule, wait max 60s
  jobTemplate:                     # same as a regular Job spec
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:15
              env:
                - name: PGPASSWORD
                  value: "mypassword"
              command:
                - sh
                - -c
                - |
                  echo "Starting DB backup at $(date)"
                  pg_dump -h postgres-service -U admin mydb > /backup/db.sql
                  echo "Backup complete!"
```

---

### 🔷 ConcurrencyPolicy Options

```
Allow    → multiple job instances can run simultaneously (default)
           Use when: jobs are independent, stateless

Forbid   → skip new job if previous one still running
           Use when: backup jobs (don't want 2 backups at once)

Replace  → cancel running job, start fresh one
           Use when: data sync (only latest run matters)
```

---

### 🛠️ CronJob Commands

```bash
# Apply
kubectl apply -f cronjob.yaml

# List cronjobs
kubectl get cronjobs -n dev
# NAME        SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE
# db-backup   0 0 * * *   False     0        <none>

# Manually trigger a CronJob right now (for testing) 🔥
kubectl create job manual-backup \
  --from=cronjob/db-backup -n dev

# See jobs created by cronjob
kubectl get jobs -n dev

# Suspend a cronjob (pause it)
kubectl patch cronjob db-backup \
  -p '{"spec":{"suspend": true}}' -n dev

# Resume it
kubectl patch cronjob db-backup \
  -p '{"spec":{"suspend": false}}' -n dev

# Delete
kubectl delete cronjob db-backup -n dev
```

---

## 🔷 PART 3 — StatefulSets

### The Problem With Deployments for Databases

```
Deployment pods are:
  ✅ Interchangeable  (any pod = any other pod)
  ✅ Stateless        (no data stored in pod)
  ❌ No stable name   (pod name changes on restart)
  ❌ No stable storage (volume detaches on reschedule)
  ❌ No ordered start/stop

This is FINE for web servers, APIs.
This is DISASTER for databases.
```

**Why databases break with Deployments:**

```
MySQL cluster needs:
  Pod 1 = PRIMARY   (handles writes)
  Pod 2 = REPLICA-1 (handles reads, replicates from primary)
  Pod 3 = REPLICA-2 (handles reads, replicates from primary)

With Deployment:
  Pod names: mysql-7d9f-abc, mysql-7d9f-def  (random!)
  Pod restarts → gets new name → replica loses primary reference
  Pod reschedules → storage detaches → DATA LOSS 💀
```

---

### 🔷 What StatefulSet Provides

```
StatefulSet gives each pod:

  1. STABLE NETWORK IDENTITY
     pod-0, pod-1, pod-2  (always same name, even after restart)

  2. STABLE STORAGE
     Each pod gets its own PersistentVolume
     Pod reschedules → same volume reattaches

  3. ORDERED DEPLOYMENT
     pod-0 starts first → pod-1 starts → pod-2 starts
     (controlled, predictable)

  4. ORDERED SCALING/DELETION
     Scale down: pod-2 dies first → pod-1 → pod-0
```

---

### 🔷 StatefulSet vs Deployment

```
┌─────────────────┬──────────────────────┬───────────────────────┐
│                 │    DEPLOYMENT        │    STATEFULSET        │
├─────────────────┼──────────────────────┼───────────────────────┤
│ Pod names       │ Random (hash suffix) │ Stable (pod-0, pod-1) │
│ Pod identity    │ Interchangeable      │ Unique, stable        │
│ Storage         │ Shared or ephemeral  │ Dedicated per pod     │
│ Start order     │ All at once          │ Sequential (0,1,2...) │
│ Scale down      │ Random pod removed   │ Reverse order (2,1,0) │
│ Use case        │ Stateless apps       │ Stateful apps         │
│ Examples        │ nginx, API, frontend │ MySQL, Kafka, MongoDB │
└─────────────────┴──────────────────────┴───────────────────────┘
```

---

### 🛠️ StatefulSet YAML — MySQL Example

```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: dev
spec:
  serviceName: "mysql"           # headless service name (required)
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "rootpassword"
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql    # mysql data directory
  volumeClaimTemplates:                    # 🔑 key difference from Deployment
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi                 # each pod gets its own 10Gi volume
```

---

### 🔷 volumeClaimTemplates — The Key Difference

```
Deployment:
  All pods share ONE volume
  OR each pod gets ephemeral storage (lost on restart)

StatefulSet with volumeClaimTemplates:
  mysql-0 → gets its own PVC: mysql-data-mysql-0 (10Gi)
  mysql-1 → gets its own PVC: mysql-data-mysql-1 (10Gi)
  mysql-2 → gets its own PVC: mysql-data-mysql-2 (10Gi)

  Pod restarts → same PVC reattaches → DATA SAFE ✅
  Pod reschedules to diff node → PVC follows → DATA SAFE ✅
```

---

### 🔷 Headless Service — Required for StatefulSet

StatefulSet needs a **Headless Service** to give each pod a stable DNS name:

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql              # must match StatefulSet's serviceName
  namespace: dev
spec:
  clusterIP: None          # ← this makes it headless (no load balancing)
  selector:
    app: mysql
  ports:
    - port: 3306
```

```bash
kubectl apply -f headless-service.yaml
kubectl apply -f statefulset.yaml

# Each pod gets a stable DNS name:
# mysql-0.mysql.dev.svc.cluster.local
# mysql-1.mysql.dev.svc.cluster.local
# mysql-2.mysql.dev.svc.cluster.local
#  │       │     │
#  pod     svc   namespace
```

> Now your replica can always find primary at `mysql-0.mysql.dev.svc.cluster.local` — **even after restarts.**

---

### 🛠️ StatefulSet Commands

```bash
# Apply
kubectl apply -f statefulset.yaml

# List
kubectl get statefulsets -n dev
kubectl get sts -n dev             # shorthand

# NAME    READY   AGE
# mysql   3/3     2m

# Watch ordered pod creation
kubectl get pods -n dev -w
# mysql-0   0/1  Pending   → Running    (first)
# mysql-1   0/1  Pending   → Running    (second, after mysql-0 ready)
# mysql-2   0/1  Pending   → Running    (third)

# Check persistent volume claims created
kubectl get pvc -n dev
# NAME                STATUS   VOLUME   CAPACITY
# mysql-data-mysql-0  Bound    pv-001   10Gi      ✅
# mysql-data-mysql-1  Bound    pv-002   10Gi      ✅
# mysql-data-mysql-2  Bound    pv-003   10Gi      ✅

# Scale
kubectl scale sts mysql --replicas=5 -n dev

# Delete StatefulSet (NOTE: PVCs are NOT deleted automatically)
kubectl delete sts mysql -n dev
kubectl get pvc -n dev             # PVCs still exist — data is safe
```

---

## 🔷 Full Workload Comparison

```
┌──────────────┬──────────────────────────────────────────────────┐
│  Resource    │  When to use                                     │
├──────────────┼──────────────────────────────────────────────────┤
│ Deployment   │ Stateless apps: web, API, frontend               │
│ DaemonSet    │ Per-node infra: logging, monitoring, networking  │
│ StatefulSet  │ Stateful apps: MySQL, Kafka, Redis, MongoDB      │
│ Job          │ One-time tasks: migrations, reports, processing  │
│ CronJob      │ Scheduled tasks: backups, syncs, cleanups        │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## ✅ Chapter 6 Summary

| Concept | Key Point |
|---|---|
| Job | Run to completion — retries on failure — then stops |
| CronJob | Scheduled Job — uses cron syntax |
| ConcurrencyPolicy | Allow / Forbid / Replace — controls overlapping runs |
| StatefulSet | Stable pod names, stable storage, ordered start/stop |
| volumeClaimTemplates | Each pod gets its own dedicated PVC |
| Headless Service | Required for StatefulSet DNS — `clusterIP: None` |

---

## 🧠 Interview Questions You Can Now Nail

1. **"When would you use a Job vs a Deployment?"**
   → Deployment for long-running services that should always be up. Job for tasks that run to completion — like DB migrations, report generation, batch processing.

2. **"What is a CronJob and how is it different from a Job?"**
   → CronJob is a Job with a schedule. It creates Jobs at specified intervals using cron syntax.

3. **"Why can't you use a Deployment for a database?"**
   → Pods get random names, no stable storage per pod, no ordered startup. Databases need stable identity, dedicated persistent storage, and ordered start/stop — all of which StatefulSet provides.

4. **"What is a Headless Service?"**
   → A service with `clusterIP: None`. No load balancing — instead gives each pod a stable DNS name. Required by StatefulSets for pod-to-pod communication.

5. **"What happens to PVCs when you delete a StatefulSet?"**
   → PVCs are NOT deleted. They persist to protect data. You must manually delete them.
