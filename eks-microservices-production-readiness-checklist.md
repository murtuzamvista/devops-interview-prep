# Amazon EKS Microservices Production Readiness Checklist

A comprehensive operational checklist and architectural reference for
deploying resilient, secure, and scalable microservices onto **Amazon
Elastic Kubernetes Service (EKS)**.

## Table of Contents
1. [Cluster & Infrastructure Foundation](#1-cluster--infrastructure-foundation)
2. [Networking & Traffic Management](#2-networking--traffic-management)
3. [Security & Access Control](#3-security--access-control)
4. [Microservice Resilience & Scaling](#4-microservice-resilience--scaling)
5. [CI/CD & GitOps Delivery](#5-cicd--gitops-delivery)
6. [Observability & Operations](#6-observability--operations)
7. [Quick-Reference Matrix](#7-quick-reference-matrix)

---

## 1. Cluster & Infrastructure Foundation
- [ ] **Multi-AZ High Availability:** Spread worker nodes and managed
  node groups across at least three Availability Zones (AZs) to survive
  single-zone failures.
- [ ] **Infrastructure as Code (IaC):** Provision the EKS cluster, VPC,
  IAM roles, and core add-ons declaratively using Terraform, OpenTofu,
  or AWS CDK with remote state locking.
- [ ] **Dynamic Node Autoscaling:** Configure Karpenter or the
  Kubernetes Cluster Autoscaler to dynamically provision, consolidate,
  and right-size nodes based on pending pod requirements.
- [ ] **Kubernetes & Add-on Lifecycle:** Confirm the cluster runs a
  Kubernetes version within AWS standard support and keep managed
  add-ons (`CoreDNS`, `kube-proxy`, `vpc-cni`, `aws-ebs-csi-driver`)
  patched.
- [ ] **Dedicated System Node Pool:** Isolate critical cluster add-ons
  (CoreDNS, ingress controllers, monitoring agents) onto dedicated
  system nodes using taints and tolerations so application spikes
  cannot starve control-plane dependencies.

---

## 2. Networking & Traffic Management
- [ ] **Private Subnets & API Endpoint Hardening:** Deploy worker nodes
  strictly in private subnets and restrict EKS API server access using
  private endpoints or allowlisted CIDR blocks.
- [ ] **IP Address Exhaustion Prevention:** Size VPC subnet CIDR blocks
  adequately or enable Amazon VPC CNI prefix delegation
  (`ENABLE_PREFIX_DELEGATION=true`) to support high pod density.
- [ ] **VPC Endpoints:** Provision Gateway endpoints (S3, DynamoDB) and
  Interface endpoints (ECR, STS, Secrets Manager, CloudWatch) to keep
  AWS API traffic off the public internet and reduce NAT Gateway data
  transfer costs.
- [ ] **Ingress & Automated TLS:** Deploy the AWS Load Balancer
  Controller (ALB/NLB) or NGINX Ingress integrated with AWS Certificate
  Manager (ACM) for automated TLS termination and renewal.
- [ ] **Encryption in Transit (mTLS & Service Traffic):** Enforce TLS 1.2+
  on all external ingress endpoints, enable Nitro-based in-transit
  encryption between supported EC2 nodes, and enforce mutual TLS
  (mTLS) for internal pod-to-pod traffic via a service mesh (Istio,
  Linkerd, or AWS VPC Lattice) and TLS-encrypted connections to
  external databases and caches.
- [ ] **Zero-Trust Network Policies:** Enforce pod-to-pod traffic
  isolation using Amazon VPC CNI network policies or Calico with a
  default-deny ingress posture per namespace.

---

## 3. Security & Access Control
- [ ] **Pod-Level IAM Permissions:** Use EKS Pod Identities or IAM Roles
  for Service Accounts (IRSA) to grant least-privilege AWS permissions
  per microservice; avoid attaching broad IAM policies to worker node
  roles.
- [ ] **Encryption at Rest (Storage, Secrets & Logs):** Encrypt all
  persistent storage and state at rest using customer-managed AWS KMS
  keys—including Kubernetes `etcd` Secrets envelope encryption, EBS
  root and persistent volumes (`aws-ebs-csi-driver` StorageClass), EFS
  file systems, RDS/ElastiCache datastores, ECR repositories, and S3
  telemetry/state buckets.
- [ ] **External Secrets Synchronization:** Sync runtime credentials
  dynamically from AWS Secrets Manager or Parameter Store via the
  External Secrets Operator instead of storing static secrets in Git.
- [ ] **Pod & Container Hardening:** Enforce Pod Security Standards
  (`restricted` profile), run containers as non-root
  (`runAsNonRoot: true`), set `readOnlyRootFilesystem: true`, and drop
  unnecessary Linux capabilities.
- [ ] **Container Image Scanning & Signing:** Scan images in Amazon ECR
  during CI builds, block deployments with critical/high CVEs, and
  verify image provenance using AWS Signer or Cosign.
- [ ] **Least-Privilege Kubernetes RBAC:** Map AWS IAM roles to granular
  Kubernetes `Role` and `ClusterRole` bindings; restrict
  `system:masters` / `cluster-admin` access to break-glass emergency
  workflows.

---

## 4. Microservice Resilience & Scaling
- [ ] **Accurate Resource Requests & Limits:** Define CPU and memory
  `requests` (for scheduler placement) and memory `limits` (to prevent
  noisy-neighbor OOM kills) based on load-test profiling.
- [ ] **Three-Tier Health Probes:** Implement `startupProbe`
  (slow-starting apps), `readinessProbe` (traffic routing gate), and
  `livenessProbe` (deadlock recovery without cascading restarts from
  downstream outages).
- [ ] **Graceful Shutdown Lifecycle:** Handle `SIGTERM` in application
  code and configure a `preStop` sleep hook (e.g., `sleep 5`) so
  ALB/kube-proxy endpoints deregister before the container stops
  accepting requests.
- [ ] **Topology Spread & Anti-Affinity:** Run at least 2–3 replicas per
  critical microservice and enforce `topologySpreadConstraints` across
  `topology.kubernetes.io/zone` and `kubernetes.io/hostname`.
- [ ] **Pod Disruption Budgets (PDBs):** Configure `minAvailable` or
  `maxUnavailable` PDBs on all stateless and stateful workloads to
  protect availability during node drains and cluster upgrades.
- [ ] **Horizontal & Event-Driven Autoscaling:** Configure Horizontal
  Pod Autoscaler (HPA) on CPU/memory/custom metrics, or use KEDA for
  queue-depth and event-driven scaling (SQS, Kafka).

---

## 5. CI/CD & GitOps Delivery
- [ ] **Pre-Commit & PR Secret Scanning:** Enforce local `pre-commit`
  hooks (using Gitleaks, TruffleHog, or `detect-secrets`) to block
  hardcoded credentials before `git push`, paired with required CI status
  checks and GitHub Push Protection to block pull request merges on
  secret detection.
- [ ] **Declarative GitOps Sync:** Use Argo CD or Flux to continuously
  reconcile cluster manifests from Git rather than pushing imperative
  `kubectl apply` commands from CI runners.
- [ ] **Immutable Container Tags:** Tag container images with immutable
  Git commit SHAs or semantic release versions; prohibit `:latest` tags
  in staging and production manifests.
- [ ] **Progressive Delivery & Rollbacks:** Configure controlled rolling
  updates (`maxUnavailable: 0`, `maxSurge: 25%`) or canary/blue-green
  deployments with Argo Rollouts and automated metric-based rollbacks.
- [ ] **Policy-as-Code Admission Control:** Enforce guardrails (e.g.,
  required labels, disallowed privileged pods, approved registries)
  prior to scheduling using Kyverno or OPA Gatekeeper.

---

## 6. Observability & Operations
- [ ] **OpenTelemetry Collection & Auto-Instrumentation:** Deploy the
  OpenTelemetry Collector and OTel/Dash0 Operator to collect OTLP
  metrics, logs, and traces without vendor lock-in.
- [ ] **Structured Logging & Trace Correlation:** Emit single-line JSON
  logs to `stdout`/`stderr` enriched with `trace_id`, `span_id`, and
  Kubernetes metadata alongside EKS control plane audit logs.
- [ ] **RED Metrics, Tracing & SLO Alerting:** Track Rate, Errors, and
  Duration (RED) metrics with distributed traces across service calls
  and configure symptom-based SLO burn-rate alerts.
- [ ] **Telemetry Filtering & Storage Efficiency:** Filter noisy health
  checks (`/healthz`, `/ready`) at the collector/SignalControl layer and
  enforce S3 lifecycle policies for self-hosted backends (Loki, Mimir,
  Tempo).
- [ ] **Load Testing & Chaos Validation:** Run pre-production load tests
  to validate HPA/Karpenter scaling speed and verify failover behavior
  during simulated AZ or dependency outages.
- [ ] **Operational Runbooks & Cost Tagging:** Maintain documented
  incident response runbooks per service and enforce Kubernetes
  labels/AWS tags for FinOps cost allocation (Kubecost or OpenCost).

---

## 7. Quick-Reference Matrix

| Domain | Risk Mitigated | AWS / K8s Mechanism |
| :--- | :--- | :--- |
| **Infrastructure** | Single-AZ outage & starvation | Multi-AZ Nodes, Karpenter, Taints |
| **Networking** | IP exhaustion & plaintext interception | Prefix Delegation, ACM TLS, Service Mesh mTLS |
| **Security & IAM** | Unencrypted data & privilege escalation | KMS Encryption at Rest (EBS/etcd/S3), Pod Identities |
| **Resilience** | Dropped traffic & cascade failures | `preStop`, PDBs, Topology Spread |
| **CI/CD & GitOps** | Repo secret leaks, drift & bad releases | Gitleaks / PR Checks, Argo CD, Kyverno |
| **Observability** | Blind spots & runaway spend | OpenTelemetry, RED Metrics, S3/Dash0 Filtering |
