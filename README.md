# ML Inference Pipeline: Production Patterns + FinOps Feedback Loop

A portfolio case study showing how I build **logical, measurable ML infrastructure** — not just "it works", but understanding the trade-offs, measuring what matters, and adjusting based on real usage.

---

## TL;DR

**What I built:** End-to-end inference pipeline for 3D medical imaging (nnU-Net on CBCT scans) running on AWS + Kubernetes, with scale-to-zero at both the application and GPU infrastructure layers.

**Why it matters:** This isn't "I deployed a pod and called it done." It demonstrates designing a pipeline with clear boundaries, explaining the trade-offs behind each choice, and using metrics as a feedback loop to make better decisions over time.

**Start here (artifacts first):**
- Architecture diagram: `docs/architecture/architecture.png`
- Business dashboard: `docs/screenshots/dashboard_business.png`
- Resource efficiency: `docs/screenshots/dashboard_efficiency.png`

![ML Inference Pipeline: Production Patterns + FinOps Feedback Loop](docs/architecture/architecture.png)

![AI Inference FinOps Dashboard](docs/screenshots/dashboard_business.png)

> Quick note: Numbers shown are from a demo environment — evidence of measurement-driven iteration, not production benchmarks.

---

## Scope / Status

| Component | Status | Notes |
|-----------|--------|-------|
| End-to-end pipeline (S3→SQS→KEDA→inference→S3) | ✅ Implemented | Live and processing scans |
| Scale-to-zero (app + GPU nodes) | ✅ Implemented | Validated via metrics |
| FinOps dashboards (business + technical) | ✅ Implemented | Real-time cost tracking |
| GitOps (ArgoCD) | ✅ Implemented | All configs in Git |
| Security (IRSA, namespace isolation) | ✅ Implemented | No static credentials |
| CI (build/push image) | ✅ Implemented | GitHub Actions → ECR |
| CD (deploy to cluster) | ✅ Implemented | ArgoCD syncs from Git (manual sync trigger if needed) |
| Image promotion (update tag in GitOps) | 🟡 Partial | Manual: bump image tag in GitOps repo + ArgoCD sync (can automate later) |
| Multi-AZ HA | 🔴 Demo scope | Single AZ acceptable for portfolio |
| Multi-tenant isolation | 🔴 Demo scope | Single workload pattern |
| SLO alerting / on-call | 🔴 Demo scope | Dashboards exist, no automated alerts |

---

## How This README Is Organized

Three main areas:

1. **Technical summary** — what's implemented end-to-end  
2. **Dashboards** — business + technical views with real metrics  
3. **ML/domain context** — why workload characteristics matter for infrastructure choices

---

## Technical Summary (What's Implemented)

### 1) End-to-end inference pipeline (live)

**The flow:**
- **S3 (input)** → **S3 Event Notifications** → **SQS (queue)** → **KEDA trigger** → **ml-worker** → **nnU-Net inference** → **S3 (output)**

Demonstrates clean batch inference with clear boundaries: storage, queueing, scaling, and execution logic each stay focused on their role.

---

### 2) App autoscaling (scale-to-zero on the worker)

The `ml-worker` sits at `replicas=0` by default. **KEDA ScaledObject** scales it **0 → N → 0** based on SQS queue depth.

No idle workers when there's nothing to process. Cold starts take 2-3 minutes (pod + node provisioning) — acceptable for batch workloads where cost control matters more than instant readiness.

---

### 3) Infrastructure autoscaling (scale-to-zero on GPU nodes)

GPU nodegroup configured with **min=0**. **Cluster Autoscaler** provisions nodes only when workers need them, removes them when idle.

GPU costs dominate the budget; scale-to-zero keeps expenses tied to actual work.

---

### 4) GPU isolation (preventing cost leakage)

GPU nodes are **gpu-only** (taints/tolerations + nodeSelector). System pods stay on CPU nodegroup.

Prevents scenarios like "Prometheus landed on GPU node and blocked scale-down."

---

### 5) Observability + FinOps metrics

- **Prometheus + Grafana** for technical metrics (latency, utilization)
- **OpenCost** for real-time cluster cost
- Business metrics: volume, P95 latency, cost/hour, unit economics

Metrics drive decisions — right-sizing, resource requests, validating scaling behavior.

---

### 6) GitOps management

**ArgoCD** manages all manifests and configs through Git. Changes via pull requests, promotion via tag bump + Sync.

---

### 7) Security / access (IRSA + AWS SSO)

**IRSA** (IAM Roles for Service Accounts) handles all runtime AWS access — no static keys in pods:
- Cluster Autoscaler: node management permissions
- KEDA: SQS read access
- ml-worker: S3 read/write + SQS consumer

**AWS SSO** is used for developer access to AWS console/CLI (separate from pipeline runtime).

Namespace isolation separates inference from system components.

---

## Design Decisions (What / Why / Trade-off)

### Scale-to-zero as the default pattern

**What:** Scale-to-zero at worker (KEDA) and GPU infrastructure (Cluster Autoscaler) layers.  
**Why:** Batch inference with unpredictable load — idle GPU time destroys unit economics.  
**Trade-off:** Cold start latency (2-3 min). For this demo, acceptable; production would evaluate warm pools, image pre-pull, or faster provisioning depending on business SLA tolerance.

---

### GPU instance choice (g5.xlarge)

**What:** **g5.xlarge** for all inference.  
**Why:** Balance of price/performance for nnU-Net 3d_fullres — enough VRAM (24GB), reasonable $/hour, acceptable latency.  
**Trade-off:** Larger instances faster but cost increase must justify throughput gains.

---

### CPU right-sizing after measurement (t3.medium → t3.large)

**What:** Started with multiple `t3.medium` nodes; after measuring real usage, consolidated to `t3.large`.  
**Why:** Shows measurement-driven iteration — guess, build, observe, adjust. CPU efficiency improved from ~6% to **33.9%** after right-sizing.  
**Trade-off:** Fewer nodes = less redundancy (fine for demo, production considers HA).

---

## Dashboards (Business + Technical Views)

### 1) Business dashboard (AI Inference FinOps)

**Current metrics:**
- **Total Scans:** 27 processed
- **P95 processing time:** 53.5s
- **Real-Time Cost:** $0.0864/h (CPU-only baseline when GPU idle)
- **Unit Economics:** $0.311/scan

**Why this matters:** Product teams need answers like "How much does one scan cost?" and "What's our burn rate if volume grows 10×?" Unit economics of **$0.311/scan** breaks down infrastructure cost per business transaction — the metric that actually matters for pricing and margin analysis.

---

### 2) Resource efficiency dashboard (ML FinOps)

**Current efficiency scores:**
- **CPU Efficiency:** 33.9% (usage vs requested)
- **RAM Efficiency:** 55.1% (usage vs requested)
- **GPU Saturation:** 0% (confirming scale-to-zero working correctly)

**Resource details:**
- CPU: 1.93 cores capacity, 1.36 cores requested, 0.460 cores real usage
- RAM: 6.91 GiB capacity, 3.23 GiB requested, 1.78 GiB real usage

**What this shows:** CPU efficiency at **33.9%** (up from ~6% pre-optimization) proves the t3.large consolidation worked. RAM efficiency at **55.1%** indicates healthy headroom without over-provisioning. GPU at **0%** during idle periods confirms scale-to-zero functions as designed.

---

## ML / 3D Computer Vision Context

Used **3D CBCT segmentation (jaw/teeth)** as workload — realistic and infrastructure-intensive.

**What I worked with:**
- nnU-Net v2 (3d_fullres): training, inference, evaluation
- 2D U-Net prototyping: understanding 2D vs 3D pipeline differences
- TTA: quality vs 3-5× slower inference trade-off
- Post-processing: mask cleaning, island removal, artifact cleanup
- Error analysis: FP/FN patterns, regression detection
- Metrics: Dice, HD95 interpretation
- Tooling: 3D Slicer for QC and validation

**Why this context matters:** Understanding that 3D models are memory-hungry shaped GPU instance selection. Knowing TTA is a business decision (quality vs cost/time) affects SLA planning. Post-processing adds latency that shows up in the **53.5s P95** metric. This domain knowledge drives better infrastructure choices.

---

## Repo Map

This README lives in the portfolio hub. Implementation split across repos:

- **GitOps repo** — ArgoCD apps, K8s manifests, KEDA configs  
- **Serving repo** — Inference worker, Dockerfile, model execution  
- **IaC repo** — Terraform (EKS, nodegroups, IRSA)  

> Some repos may be private; this portfolio repo stays public with sanitized excerpts (YAML snippets, screenshots, diagrams).

---

## What This Is (and What It's Not)

**This is:**
- Portfolio case study showing end-to-end ML infrastructure thinking
- Working pipeline with real autoscaling, observability, and measurable iteration
- Evidence of design → measure → adjust methodology

**This is not:**
- Full production system (see Scope/Status table above)
- Benchmark report — numbers are demo evidence, not performance guarantees
- One-size-fits-all solution — choices made for this specific workload

---

## Appendix: Measurement Evidence

**CPU right-sizing impact:**
- Efficiency: ~6% → **33.9%** (dashboard verified)
- Configuration: 3× t3.medium → 1× t3.large
- Cost: $0.1296/h → $0.0864/h (-33%)

**Current resource allocation:**
- CPU: 1.36 cores requested, 0.460 cores real usage → room for optimization
- RAM: 3.23 GiB requested, 1.78 GiB real usage → healthy headroom

**Unit economics breakdown:** $0.311/scan represents full infrastructure cost per business transaction at current scale (27 scans processed)

![ML FinOps Resource Efficiency Dashboard](docs/screenshots/dashboard_efficiency.png)
