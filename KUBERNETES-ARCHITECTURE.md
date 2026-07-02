# Kubernetes Architecture — Video Streaming Platform

A deep dive into **how Kubernetes is used** across the `videostreamingplatform*` repos,
focused on **networking (ingress/egress, DNS, NAT), computation (Deployments /
StatefulSets / scaling), and storage (persistence + resiliency)**.

> **Rendered diagrams:** see [`KUBERNETES-ARCHITECTURE-diagrams.md`](./KUBERNETES-ARCHITECTURE-diagrams.md)
> for Mermaid versions of the network topology, workload/data-flow, ingress/egress and
> resiliency views (GitHub renders them inline). The ASCII diagrams below remain the
> terminal-friendly reference.

This is a companion to [`KUBERNETES.md`](./KUBERNETES.md) (GitOps/ArgoCD/CI/RBAC and
manifest ownership). Where the two overlap, this document is the authoritative source
for topology, sizing and networking — it reflects the current 3-broker Kafka, the
userservice/notifications additions, NLB (not NodePort) external access on AWS, and the
per-AZ NAT / EBS-CSI-pinning story.

Everything here is sourced from live manifests and Terraform:
- `videostreamingplatform/k8s/aws/terraform/` — VPC, EKS, RDS, S3, CloudFront, Redis, IRSA
- `videostreamingplatform/k8s/aws/manifests/` — core Go services (canonical AWS deploy path)
- `videostreamingplatform/k8s/local/` — Kind manifests
- `videostreamingplatform-infra/{kafka,pgvector,elasticsearch,networking,charts}/` — data plane + service charts

---

## 1. Two clusters, one topology

| | **Local (Kind)** | **AWS dev (EKS)** | **AWS prod (EKS)** |
|---|---|---|---|
| Provisioner | `k8s/local/terraform/kind.tf` | `k8s/aws/terraform` (`terraform.dev.tfvars`) | `k8s/aws/terraform` (`terraform.prod.tfvars`) |
| K8s version | 1.32.2 | 1.30 | 1.32 |
| Nodes | 1 control-plane + 1 worker | `t3.micro` × min 10 / desired 10 / max 20 | `t3.medium` × min 3 / desired 5 / max 10 |
| AZs / subnets | n/a | 2 AZs (2 public + 2 private /24) | 3 AZs (3 public + 3 private /24) |
| DB | MySQL Deployment + PVC | RDS `db.t3.micro`, single-AZ | RDS `db.t3.medium`, **multi-AZ** |
| Object store | MinIO Deployment | S3 (`aws:kms`, versioned) | S3 (`aws:kms`, versioned) |
| Cache | Redis Deployment | ElastiCache `cache.t3.micro` × 1 | ElastiCache `cache.r6g.large` × 2 (failover) |
| Search | ES StatefulSet in-cluster | ES StatefulSet in-cluster | ES StatefulSet in-cluster |
| CDN | cdn-proxy Deployment | CloudFront `PriceClass_100` | CloudFront `PriceClass_200` |
| External access | NodePort + Kind host port-maps | NLB per service | NLB per service |

> The data plane (Kafka, pgvector, Elasticsearch) runs **as in-cluster StatefulSets in
> every environment** — no MSK, no OpenSearch domain, no Aurora. Only S3, RDS,
> ElastiCache, CloudFront and Glue/Athena are AWS-managed. This keeps topology
> identical across Kind and EKS.

---

## 2. AWS network topology (VPC, subnets, IGW, NAT)

```
                                   Internet
                                       │
              ┌────────────────────────┼─────────────────────────┐
              │                        │                          │
        ┌─────▼──────┐         ┌───────▼────────┐         (viewers) HTTPS
        │  NLB(s)    │         │ Internet GW    │                 │
        │ per Svc    │         │ (aws_igw.main) │           ┌─────▼──────┐
        └─────┬──────┘         └───────┬────────┘           │ CloudFront │
              │                        │                     │ (OAI→S3)   │
 ═════════════│════════ VPC 10.0.0.0/16 (DNS on) ═══════════│════════════│═══════
              │                        │                     └─────┬──────┘
   ┌──────────┴─────────── PUBLIC subnets (map_public_ip) ────────┴────────┐
   │  10.0.1.0/24 (AZ-a)          10.0.2.0/24 (AZ-b)      [10.0.3.0/24 prod]│
   │   ├─ NLB ENIs                                                          │
   │   └─ NAT GW #1 (+EIP)         NAT GW #2 (+EIP)        [NAT GW #3 prod] │
   │   route: 0.0.0.0/0 → IGW  (aws_route_table.public)                    │
   └──────────┬──────────────────────┬─────────────────────┬──────────────┘
              │  (egress only)        │                     │
   ┌──────────▼─────────── PRIVATE subnets ────────────────▼──────────────┐
   │  10.0.11.0/24 (AZ-a)        10.0.12.0/24 (AZ-b)   [10.0.13.0/24 prod] │
   │   ┌──────────────────────────────────────────────────────────────┐   │
   │   │  EKS worker nodes (managed node group, private only)          │   │
   │   │   • all application pods                                      │   │
   │   │   • Kafka / pgvector / ES StatefulSets (on ebs-csi nodes)     │   │
   │   └──────────────────────────────────────────────────────────────┘   │
   │   RDS MySQL (db subnet group)   ElastiCache Redis (subnet group)      │
   │   route: 0.0.0.0/0 → NAT GW (per-AZ, aws_route_table.private[*])      │
   └───────────────────────────────────────────────────────────────────────┘
```

**Key facts**

- **One VPC** `10.0.0.0/16`, `enable_dns_hostnames` + `enable_dns_support` on.
- **Public subnets** (`map_public_ip_on_launch = true`) hold only the **NAT gateways**
  and the **NLB ENIs**. Route table: `0.0.0.0/0 → Internet Gateway`.
- **Private subnets** hold **everything else** — EKS nodes/pods, RDS, ElastiCache.
  No node has a public IP.
- **NAT: one gateway per private subnet (per AZ)** — `count = length(private_subnet_cidrs)`,
  each with a dedicated EIP, placed in the matching public subnet. Each private subnet's
  route table sends `0.0.0.0/0` to *its own AZ's* NAT GW. So **dev = 2 NAT GWs, prod = 3**
  — no cross-AZ NAT data-transfer, and an AZ's egress survives another AZ's NAT failure.
- **EKS node group lives only in private subnets** (`subnet_ids = aws_subnet.private[*].id`);
  the cluster control-plane ENIs span both private + public.

---

## 3. Ingress (external access)

**There is no Ingress controller anywhere** (confirmed: no `kind: Ingress` in any repo).
External access is deliberately per-service load balancing:

| Environment | Mechanism | Detail |
|---|---|---|
| **AWS** | `Service type: LoadBalancer` → **NLB** | Annotated `aws-load-balancer-type: nlb` + `cross-zone-load-balancing-enabled: true`. One NLB per public service. |
| **Local** | `Service type: NodePort` + Kind `extra_port_mappings` | NodePort on the Kind node, host-mapped to a localhost port. |
| **Control plane** | EKS API endpoint | `endpoint_public_access = true` **and** `endpoint_private_access = true`; cluster SG allows `443` from `0.0.0.0/0`. |

**Externally reachable services**

| Service | Container port | AWS (NLB) | Local NodePort → host |
|---|---|---|---|
| metadata-service | 8080 | NLB :8080 | 30080 → localhost:8080 |
| data-service | 8081 | NLB :8081 | 30081 → localhost:8081 |
| user-service | 8082 | NLB :8082 | 30082 → localhost:8082 |
| notifications | 8083 | NLB :8083 | 30093 |
| recommendation-api | 8000 | **ClusterIP only** (internal; proxied via metadata-service `/recommendations`) | ClusterIP |

**Video download path bypasses the cluster for bytes**: viewers hit **CloudFront**, which
reads objects from the S3 videos bucket via an **Origin Access Identity** (bucket is fully
private — `block_public_acls`/`restrict_public_buckets` all true). `viewer_protocol_policy =
redirect-to-https`. The `cdn-invalidator` Deployment consumes `video-events` from Kafka and
issues CloudFront invalidations when content changes.

---

## 4. Egress & security groups (the SG chain)

All outbound from private subnets goes **node → per-AZ NAT GW → IGW → Internet**
(image pulls from ghcr.io, Bedrock, AWS APIs, Razorpay, etc.).

Security-group ingress rules form a tight chain (all egress is open `-1`/`0.0.0.0/0`):

```
  0.0.0.0/0 ──443──► cluster SG (EKS API)
  cluster SG ──all tcp──► nodes SG          (control-plane → kubelet)
  VPC CIDR   ──all tcp──► nodes SG          (pod-to-pod / intra-VPC)
  nodes SG + cluster SG ──3306──► RDS SG     (MySQL)
  nodes SG ──6379──► Redis SG                (ElastiCache)
  nodes SG ──443──► OpenSearch SG            (pre-created; domain not provisioned)
```

RDS and Redis are only reachable **from the node SG** — never from the internet
(`publicly_accessible = false`, no public subnet placement).

**VPC CNI prefix delegation**: the `vpc-cni` addon runs with
`ENABLE_PREFIX_DELEGATION=true` / `WARM_PREFIX_TARGET=1`. On `t3.micro` this raises the
per-node pod cap from **4 → ~110** by assigning `/28` prefixes to ENIs — the single change
that makes free-tier nodes usable for real workloads.

---

## 5. In-cluster DNS & service discovery

Pods talk to each other over **stable cluster DNS** (`<svc>.<ns>.svc.cluster.local`),
identical in Kind and EKS:

| DNS name | Port | Purpose |
|---|---|---|
| `kafka.infra.svc.cluster.local` | 9092 | Kafka bootstrap (headless Service) |
| `kafka-{0,1,2}.kafka.infra.svc.cluster.local` | 9092/9093 | Per-broker stable identity (KRaft quorum) |
| `pgvector.infra.svc.cluster.local` | 5432 | Recommendations + weekly-report DB |
| `elasticsearch.videostreamingplatform.svc.cluster.local` | 9200 | Search index |
| `user-service.videostreamingplatform.svc.cluster.local` | 8082 | Payment link `PUBLIC_BASE_URL` |
| `jaeger.observability.svc.cluster.local` | 4318 | OTLP-HTTP traces |
| `localstack.infra.svc.cluster.local` | 4566 | Glue schema registry (local only) |

- **Kafka uses a headless Service** (`clusterIP: None`) so clients resolve individual
  broker pod DNS names — required for the KRaft controller quorum
  (`KAFKA_CONTROLLER_QUORUM_VOTERS = 1@kafka-0…,2@kafka-1…,3@kafka-2…`).
- **Network policies** (in `infra` + `videostreamingplatform` namespaces) restrict the
  data plane to same-platform callers: each policy admits ingress only `from`
  `namespaceSelector: app.kubernetes.io/part-of=videostreamingplatform` on the specific
  port — Kafka 9092, pgvector 5432, Elasticsearch 9200, LocalStack 4566.

---

## 6. Computation — stateless vs stateful

### 6a. Deployments (stateless)

All request-serving and consumer workloads are stateless Deployments; state lives in
RDS/S3/Kafka/pgvector/ES. Core Go services run `readOnlyRootFilesystem`, `runAsNonRoot`,
`allowPrivilegeEscalation: false`.

| Workload | Repo / owner | Replicas (AWS) | Resources (req → lim) | Notes |
|---|---|---|---|---|
| metadata-service | main / manifests | 2 (HPA 2–10) | 200m/256Mi → 1000m/1Gi | RollingUpdate maxSurge 0 / maxUnavailable 1; soft pod anti-affinity |
| data-service | main / manifests | 2 (HPA 2–10) | 200m/256Mi → 1000m/1Gi | Same strategy; JWT-gated download |
| user-service | main / manifests | 1 (no HPA) | 100m/128Mi → 500m/384Mi | Auth/subscriptions/payments |
| cdn-invalidator | main / manifests | 1 | 50m/64Mi → 200m/128Mi | Kafka `video-events` → CloudFront invalidation |
| recommendation-api | reco / Helm chart | 2 (HPA 2–6) | 200m/256Mi → 1000m/512Mi | FastAPI; ClusterIP only |
| notifications | notif / Helm chart | 1 | (chart) | WS gateway (NLB on AWS); Redis backplane; `subscription-events` → `notification-events` |
| kafka-es-consumer | analytics / Helm | 1 | 100m/128Mi → 500m/256Mi | `video-events` → Elasticsearch |
| watch-history-consumer | analytics / Helm | 1 | 100m/256Mi → 250m/512Mi | `watch-events` → Iceberg (Glue+S3) |

### 6b. StatefulSets (stateful)

| StatefulSet | Namespace | Replicas | PVC (per pod) | Placement | Resiliency |
|---|---|---|---|---|---|
| **kafka** | infra | **3** | `kafka-data` 5Gi RWO | `nodeSelector ebs-csi=true` + toleration; **required** pod anti-affinity (one broker per node); `podManagementPolicy: Parallel` | RF=3, offsets/txn RF=3, `min.insync.replicas=2` → survives 1 broker/node loss |
| **pgvector** | infra | **1** | `pgvector-data` 5Gi RWO | ebs-csi node | **SPOF** — reschedules on node loss but EBS is AZ-bound |
| **elasticsearch** | videostreamingplatform | **1** | `es-data` 10Gi RWO | ebs-csi node | **SPOF** — `discovery.type=single-node` |

`podManagementPolicy: Parallel` on Kafka is required for KRaft bootstrap: with the default
`OrderedReady`, `kafka-0` can't become Ready until the quorum forms (needs 2/3 voters), but
`kafka-1` wouldn't start until `kafka-0` is Ready — a deadlock. Parallel lets all three boot
and elect a leader.

### 6c. Jobs & CronJobs

- **Jobs (one-shot)**: `db-init` (RDS schema), `kafka-topics-init` (creates
  `video-events` 3p, `watch-events` 6p, `subscription-events` 3p, `notification-events` 3p,
  `notification-events-dlq` 1p), analytics `catalog-init` (Iceberg tables), MinIO
  `bucket-init` (local).
- **CronJobs**: analytics `trending` + `user-features` + Iceberg compaction; notifications
  `weekly-report` (reads pgvector).

### 6d. Scaling

- **Horizontal Pod Autoscalers** (`autoscaling/v2`): metadata & data services 2→10 @ CPU 70% /
  mem 80%; recommendations 2→6 @ CPU 60%. Tuned behavior: scale-up 60s window (+2 pods/60s),
  scale-down 300s window (−1 pod/120s) to damp flapping.
- **Cluster/node scaling**: EKS managed node group `min/desired/max` (dev 10/10/20,
  prod 3/5/10). StatefulSets do **not** autoscale (fixed replicas).

### 6e. Scheduling — EBS-CSI node pinning (free-tier design)

The EBS CSI driver is installed via the **upstream Helm chart** (not the EKS addon) so the
node DaemonSet's `nodeSelector` can be pinned. StatefulSet-hosting nodes are labeled
`ebs-csi=true` and tainted `dedicated=ebs-csi:NoSchedule`; the `ebs-csi-node` DaemonSet and
the three StatefulSets select+tolerate those nodes, so CSI pods don't fan out to every node
and general workloads don't squat on the storage nodes. The CSI controller SA uses IRSA role
`ebs-csi-irsa-<env>` (`AmazonEBSCSIDriverPolicy`).

---

## 7. Storage & resiliency

### 7a. Persistent volumes (in-cluster)

- All three StatefulSets use `volumeClaimTemplates`, `ReadWriteOnce`, default **gp3**
  StorageClass (`WaitForFirstConsumer`, encrypted).
- **`WaitForFirstConsumer` pins each PV to the AZ of the node that first scheduled the pod.**
  This is the crux of the SPOF story below: an EBS volume cannot cross AZs, so a pod that
  reschedules into a *different* AZ cannot reattach its volume.

### 7b. AWS-managed storage

| Resource | Encryption | Versioning / backup | Resiliency |
|---|---|---|---|
| **RDS MySQL** | KMS at rest | dev backup 1d + skip final snapshot; prod backup 7d + final snapshot + deletion protection | **dev single-AZ, prod multi-AZ**; master password managed by AWS in Secrets Manager |
| **S3 videos** | SSE-KMS + bucket keys | versioning on; lifecycle: retention (dev 30d/prod 365d), noncurrent expire 30d, abort incomplete multipart 7d | 11-nines durability; full public-access block; CloudFront OAI read only |
| **ElastiCache Redis** | at-rest on (transit off) | dev no snapshot; prod snapshot retention 3 | dev 1 node; prod 2 nodes + `automatic_failover` |

### 7c. Failure-mode matrix (the "resiliency cases")

| Failure | Blast radius | Behavior |
|---|---|---|
| **Stateless pod dies** | none | Deployment reschedules; NLB/Service drops it via readiness probe |
| **Single node lost** | pods on it | Rescheduled elsewhere; HPA/node group backfills. Stateful pod recovers **only if it lands in the same AZ** (EBS reattach) |
| **One Kafka broker/node lost** | none | RF=3 + `min.insync.replicas=2` keep topics writable; anti-affinity guarantees the other 2 brokers are on other nodes |
| **AZ lost** | that AZ's NAT + nodes + any EBS-bound stateful pod there | Other AZs keep egress (per-AZ NAT) and serving; **pgvector/ES/single-AZ RDS in the lost AZ are down until AZ returns** |
| **pgvector or Elasticsearch down** | recommendations / search degraded | **SPOF (1 replica each)** — no HA; reschedule + EBS reattach is the only recovery. Documented tradeoff for cost |
| **RDS failover (prod)** | brief write pause | Multi-AZ standby promoted automatically |
| **S3 object churn** | none | Versioning + noncurrent retention allow recovery; abandoned multipart parts reclaimed after 7d |

> **Known SPOFs by design (cost tradeoff):** pgvector and Elasticsearch run single-replica.
> They survive process/node loss via reschedule **within the same AZ**, but have no
> cross-AZ HA. Kafka (RF=3) and prod RDS (multi-AZ) are the only genuinely HA stores.

---

## 8. Identity & secrets on the network path

- **IRSA** (IAM Roles for Service Accounts) via the cluster OIDC provider — no static AWS
  keys in pods:
  - `metadata-service` SA → `rds-db:connect`
  - `data-service` SA → S3 (`Get/Put/Delete/List`) + KMS (`Decrypt`/`GenerateDataKey`)
  - `ebs-csi-controller-sa` (kube-system) → `AmazonEBSCSIDriverPolicy`
  - `opensearch-irsa` maps analytics + recommendations SAs (reserved; ES is in-cluster today)
- **`aws-auth` ConfigMap** maps the node role (`system:nodes`), the GitHub Actions deploy
  role and account root to `system:masters`.
- **Cross-cutting secrets**: `rds-credentials` (from Secrets Manager), `auth-secrets`
  (`JWT_SIGNING_SECRET`, shared by userservice signer + dataservice verifier),
  `razorpay-credentials`, `ghcr-pull` (image pulls). `app-config` ConfigMap carries the
  non-secret env (endpoints, TTLs, topic names) via `envFrom`.

---

## 9. Manifest ownership (where to change what)

| Concern | Location |
|---|---|
| VPC / subnets / NAT / IGW / SGs / EKS / IRSA / RDS / S3 / CloudFront / Redis | `videostreamingplatform/k8s/aws/terraform/` |
| Core Go services (Deploy/Svc/HPA) on AWS | `videostreamingplatform/k8s/aws/manifests/` (canonical `kubectl` path) |
| Local Kind manifests + `kind.tf` port maps | `videostreamingplatform/k8s/local/` |
| Kafka / pgvector / ES StatefulSets, namespaces, network policies | `videostreamingplatform-infra/{kafka,pgvector,elasticsearch,networking}/` |
| analytics / recommendations / notifications Helm charts | `videostreamingplatform-infra/charts/` |
| EBS-CSI install + node labels/taints | `videostreamingplatform-infra/scripts/bootstrap-aws.sh` |

> Note: `videostreamingplatform/k8s/aws/helm/videostreamingplatform/` exists as an
> alternative chart but is **not** the path `bootstrap-aws.sh` / `deploy-platform.yml` use —
> they apply the raw `manifests/` for core services and Helm only for ebs-csi, analytics,
> recommendations and notifications.
