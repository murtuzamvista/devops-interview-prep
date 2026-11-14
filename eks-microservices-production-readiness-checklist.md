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
  or AWS CDK.
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
- [ ] **Zero-Trust Network Policies:** Enforce pod-to-pod traffic
  isolation using Amazon VPC CNI network policies or Calico with a
  default-deny ingress posture per namespace.

---

## 3. Security & Access Control
- [ ] **Pod-Level IAM Permissions:** Use EKS Pod Identities or IAM Roles
  for Service Accounts (IRSA) to grant least-privilege AWS permissions
  per microservice; avoid attaching broad IAM policies to worker node
  roles.
- [ ] **Secrets Encryption & External Sync:** Enable envelope encryption
  for Kubernetes Secrets using a customer-managed AWS KMS key and sync
  credentials dynamically from AWS Secrets Manager via the External
  Secrets Operator.
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
- [ ] **Centralized Structured Logging:** Emit JSON logs to
  `stdout`/`stderr` and forward application logs plus EKS control plane
  audit logs to CloudWatch Logs, OpenSearch, or Datadog via Fluent Bit.
- [ ] **Metrics & Actionable Alerting:** Scrape cluster and service
  metrics with Prometheus or Amazon Managed Service for Prometheus;
  alert on SLO burn rates, 5xx spikes, `CrashLoopBackOff`, and node
  disk/memory pressure.
- [ ] **Distributed Tracing:** Instrument services with OpenTelemetry
  (OTel) and propagate trace context headers across HTTP/gRPC and async
  messaging boundaries.
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
| **Networking** | IP exhaustion & exposed APIs | Prefix Delegation, Private Endpoints |
| **Security & IAM** | Credential leaks & escalation | Pod Identities / IRSA, KMS Secrets |
| **Resilience** | Dropped traffic & cascade failures | `preStop`, PDBs, Topology Spread |
| **CI/CD & GitOps** | Config drift & bad releases | Argo CD, SHA Tags, Kyverno |
| **Observability** | Blind spots & runaway spend | Fluent Bit, Prometheus/OTel, OpenCost |

