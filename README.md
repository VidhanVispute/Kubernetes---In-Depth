Immediate (quick wins):
  → TCS, Infosys, Wipro (DevOps roles) — ₹8-10 LPA
  → Capgemini, HCL Tech DevOps — ₹8-12 LPA
  → Mid-size IT services

Target (good companies):
  → Razorpay, CRED, Groww (fintech)
  → Swiggy, Zomato (product)
  → Freshworks, Zoho (SaaS)
  → Atlassian, Walmart Global Tech
  → Any Series B/C startup

Platforms to use:
  → LinkedIn (most important — update TODAY)
  → Naukri.com (still works in India)
  → AngelList / Wellfound (startups)
  → Instahyre (product companies)
  → Referrals (most underused — reach out to college contacts)



  # 📊 Roadmap vs What We Covered — Honest Analysis

---

## ✅ What We FULLY Covered

```
PHASE 1 — Kubernetes Core Fundamentals
  ✅ Full architecture deep dive (API, etcd, scheduler,
     controller manager, kubelet, kube-proxy)
  ✅ kubectl apply → API → etcd → controller →
     scheduler → kubelet → container flow
  ✅ Reconciliation loop — explained deeply
  ✅ Pod, ReplicaSet, Deployment, StatefulSet,
     DaemonSet, Job, CronJob, Namespace
  ✅ Labels & Selectors
  ✅ ClusterIP, NodePort, LoadBalancer,
     Headless Service, Ingress, CoreDNS
  ✅ PV, PVC, StorageClass, Dynamic provisioning
  ✅ Volume binding modes
  ✅ StatefulSets with PVC

PHASE 2 — Production Engineering
  ✅ Taints & Tolerations
  ✅ Node Affinity + Pod Affinity
  ✅ Resource requests & limits
  ✅ QoS classes
  ✅ Liveness, Readiness, Startup probes
  ✅ HPA + VPA + Metrics Server
  ✅ Cluster Autoscaler
  ✅ RBAC deeply
  ✅ Service Accounts
  ✅ Prometheus + Grafana + Loki + Alertmanager
  ✅ IRSA (in EKS project)

PHASE 3 — Advanced
  ✅ CRDs & Operators (cert-manager, Strimzi)
  ✅ Helm — chart structure, values,
     templating, lifecycle
  ✅ Service Mesh — Istio fully covered
     (mTLS, traffic routing, canary,
      circuit breaking, fault injection)
  ✅ Sidecar + Init containers

PHASE 4 — EKS Production
  ✅ EKS deep — control plane, managed
     node groups, OIDC, IRSA
  ✅ ALB Ingress Controller
  ✅ Multi-AZ design
  ✅ 3-tier production deployment
  ✅ HPA + PVC + Secrets + RBAC + Monitoring
  ✅ Zero-downtime deployments
  ✅ Blue-Green strategy
  ✅ ArgoCD GitOps

PHASE 6 — DevOps Integration
  ✅ GitHub Actions CI/CD
  ✅ ArgoCD (GitOps)
  ✅ Terraform mentioned (doing next)
```

---

## ⚠️ What We PARTIALLY Covered

```
PHASE 0 — Prerequisites
  ⚠️ Linux internals (cgroups, namespaces)
     → mentioned conceptually
     → NOT deep dived

  ⚠️ Container runtime (containerd)
     → mentioned
     → didn't install manually

  ⚠️ Networking deep (iptables, NAT,
     TCP handshake, OSI model)
     → mentioned in kube-proxy context
     → NOT standalone deep dive

PHASE 2 — Security
  ⚠️ Pod Security Standards
     → NOT covered
  ⚠️ Network Policies (Calico)
     → mentioned briefly
     → NOT hands-on

  ⚠️ Secrets encryption at rest
     → mentioned as best practice
     → NOT configured hands-on
```

---

## ❌ What We Did NOT Cover

```
PHASE 3 — Advanced
  ❌ etcd deep dive
     → backup & restore
     → leader election
     → certificate rotation
     → "break etcd and restore" exercise

  ❌ CNI Plugins (Calico manual install)
     → pod networking internals
     → IP allocation
     → IPIP / VXLAN

PHASE 4 — EKS
  ❌ Fargate vs EC2 (deep comparison)
  ❌ Cluster version upgrade
  ❌ Node group rotation
  ❌ Backup with Velero
  ❌ Disaster recovery simulation

PHASE 5 — Failure Engineering
  ❌ Intentional chaos testing
     → kill nodes
     → break DNS
     → block network
     → fill disk
     → overload memory
  (This entire phase not covered)
```

---

## 🔷 Honest Scorecard

```
Phase 0  — Prerequisites        40% covered
Phase 1  — Core Fundamentals    95% covered ✅
Phase 2  — Production Skills    80% covered ✅
Phase 3  — Advanced K8s         75% covered ✅
Phase 4  — EKS Production       85% covered ✅
Phase 5  — Failure Engineering   0% covered ❌
Phase 6  — DevOps Integration   80% covered ✅

Overall: ~75-80% of full roadmap covered
```

---

## 🔷 What This Means For You

```
Good news:
  The 20-25% we missed is NOT blocking
  you from getting a job.

  What we covered is MORE than enough
  for 90% of DevOps interviews.

The missing pieces matter for:
  → Senior/Staff level roles (₹35 LPA+)
  → Companies doing serious SRE work
  → CKA exam (etcd backup is tested!)

Priority order to fill gaps:
  1. etcd backup/restore     (CKA critical)
  2. Network Policies        (security interviews)
  3. Calico CNI basics       (networking interviews)
  4. Failure engineering     (SRE roles)
  5. Linux cgroups deep      (fundamentals interviews)
```

---

## 🔷 Recommendation

```
For your immediate goal (get ₹14-18 LPA job):
  → What we covered is ENOUGH
  → Focus on building projects NOW
  → Fill gaps AFTER you're employed

For CKA certification:
  → Add etcd backup/restore (3 days)
  → Add Network Policies (2 days)
  → Then you're CKA ready

For long term (₹30 LPA+):
  → Complete Phase 5 (failure engineering)
  → That's what makes you elite
```

---

**Bottom line — we covered the right 80%.
The remaining 20% won't stop you from
landing your next role. Build the projects
first. Fill gaps in parallel. 🚀**
