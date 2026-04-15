# Scale Roadmap — 1K to 1M req/s

This document captures the architectural decisions and rationale for scaling the video streaming platform from ~200 req/s to 1M req/s.

---

## Scale Tiers

| Scale | Architecture | Key Changes |
|-------|-------------|-------------|
| 1K req/s | Single cluster, namespace isolation | HPA, connection pooling, managed OpenSearch |
| 10K req/s | Single cluster + caching + CDN | Redis, rate limiting, CDN for video delivery |
| 100K req/s | Multi-cluster, regional | Serving/data cluster split, KEDA, service mesh |
| 1M req/s | Multi-region, cell-based | Independent cells per region, global routing |

---

## Current Architecture Ceiling

See [SCALE.md](SCALE.md) for component-level analysis. Current ceiling: ~200–500 req/s sustained, limited by no caching, no CDN, and single-instance databases.

---

## Why Single Cluster First (Not Multi-Cluster from Day 0)

### Multi-cluster from day 0 would cost more and teach less

**Cost of premature multi-cluster (5 namespaces → 5 clusters):**

| Concern | Single Cluster | 5 Clusters |
|---------|---------------|------------|
| EKS control plane cost | $73/month | $365/month (idle) |
| VPC peering connections | 0 | 10 (every cluster talks to 2+ others) |
| ArgoCD registrations | 1 | 5 (+ RBAC per cluster) |
| IAM OIDC providers (IRSA) | 1 | 5 |
| Certificate management | 1 | 5 (mTLS between clusters) |
| Cross-service latency | ~0ms (cluster DNS) | 1-5ms (VPC peering hop) |
| Debugging complexity | Traces in one cluster | Traces span cluster boundaries |

### Migration from single to multi-cluster is NOT a rewrite

| Layer | Migration Effort | Why |
|-------|-----------------|-----|
| Application code | **Zero** | Services read env vars, don't know they're in K8s |
| Service discovery | **ConfigMap swap** | `kafka.infra.svc.cluster.local` → MSK endpoint |
| K8s manifests | **Copy + adjust** | Same Deployments, different target cluster |
| Terraform | **Moderate** | Add second `aws_eks_cluster`, split node groups |
| ArgoCD | **Moderate** | Register additional cluster, split ApplicationSets |
| Networking | **Most work** | VPC peering or Transit Gateway between clusters |

### What makes migration easy (already in place)

All cross-namespace communication uses environment variables — not hardcoded DNS:

```
metadata-service  → recommendations:  RECOMMENDATION_SERVICE_URL    ✅
services          → Kafka:            KAFKA_BROKERS                 ✅
services          → MySQL:            MYSQL_HOST                    ✅
services          → Jaeger:           OTEL_EXPORTER_OTLP_ENDPOINT  ✅
services          → ES/OpenSearch:    ELASTICSEARCH_URL             ✅
recommendations   → pgvector:         PGVECTOR_URL                 ✅
```

Each namespace maps 1:1 to a future cluster. Migration is a ConfigMap change, not a code change.

### The real learning comes from hitting walls

Scaling walls teach system design; premature optimization teaches plumbing:

- "Prometheus OOMing scraping 200 pods" → learn federated Prometheus → understand why observability separates
- "Spark jobs starving API pods of CPU" → learn node isolation → understand why clusters split
- "Analytics deploy broke metadata-service" → learn blast radius → understand why multi-cluster matters

---

## One Control Plane Per Cluster — Why

Each EKS cluster has its own control plane (API server, etcd, scheduler). This is a Kubernetes design constraint — etcd stores all cluster state and two clusters can't share one etcd.

```
EKS Cluster = Control Plane + Worker Nodes

┌─ EKS Cluster 1 ──────────────────┐
│  Control Plane (AWS-managed)      │  ← $0.10/hr, cannot be shared
│  ├─ API Server                    │
│  ├─ etcd (cluster state)          │
│  ├─ Scheduler                     │
│  └─ Controller Manager            │
│  Worker Nodes (your EC2)          │
└───────────────────────────────────┘
```

**Fleet management tools** (ArgoCD, Rancher, Cluster API) sit above, not replace, control planes. They deploy to multiple clusters but each cluster is independent.

**vCluster** is the closest alternative — virtual K8s clusters inside one host cluster. Useful for dev/test cost savings, not proven for production at scale.

---

## Cell-Based Architecture (Target: 1M req/s)

At 1M req/s, the industry pattern is cell-based architecture — independent, self-contained units that each handle a shard of traffic.

```
                         ┌──────────────────────┐
                         │   Global Edge (CDN)   │
                         │  CloudFront / Akamai  │
                         │  • Video delivery     │
                         │  • Static assets      │
                         │  • ~90% of bandwidth  │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │   Global Router       │
                         │  Route53 + Global     │
                         │  Accelerator          │
                         │  • Geo-based routing  │
                         │  • User-to-cell map   │
                         └──────────┬───────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
    ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
    │   Cell US-East   │   │  Cell US-West    │   │  Cell EU-West   │
    │                  │   │                  │   │                  │
    │ ┌──────────────┐ │   │ ┌──────────────┐ │   │ ┌──────────────┐ │
    │ │ Serving      │ │   │ │ Serving      │ │   │ │ Serving      │ │
    │ │ Cluster      │ │   │ │ Cluster      │ │   │ │ Cluster      │ │
    │ │ • API GW     │ │   │ │ • API GW     │ │   │ │ • API GW     │ │
    │ │ • metadata   │ │   │ │ • metadata   │ │   │ │ • metadata   │ │
    │ │ • data-svc   │ │   │ │ • data-svc   │ │   │ │ • data-svc   │ │
    │ │ • reco-api   │ │   │ │ • reco-api   │ │   │ │ • reco-api   │ │
    │ └──────────────┘ │   │ └──────────────┘ │   │ └──────────────┘ │
    │                  │   │                  │   │                  │
    │ ┌──────────────┐ │   │ ┌──────────────┐ │   │ ┌──────────────┐ │
    │ │ Data Cluster │ │   │ │ Data Cluster │ │   │ │ Data Cluster │ │
    │ │ • Kafka      │ │   │ │ • Kafka      │ │   │ │ • Kafka      │ │
    │ │ • Analytics  │ │   │ │ • Analytics  │ │   │ │ • Analytics  │ │
    │ │ • pgvector   │ │   │ │ • pgvector   │ │   │ │ • pgvector   │ │
    │ └──────────────┘ │   │ └──────────────┘ │   │ └──────────────┘ │
    │                  │   │                  │   │                  │
    │ ┌──────────────┐ │   │ ┌──────────────┐ │   │ ┌──────────────┐ │
    │ │ Stateful     │ │   │ │ Stateful     │ │   │ │ Stateful     │ │
    │ │ • RDS primary│ │   │ │ • RDS replica│ │   │ │ • RDS replica│ │
    │ │ • S3 bucket  │ │   │ │ • S3 replica │ │   │ │ • S3 replica │ │
    │ │ • OpenSearch │ │   │ │ • OpenSearch │ │   │ │ • OpenSearch │ │
    │ └──────────────┘ │   │ └──────────────┘ │   │ └──────────────┘ │
    └─────────────────┘   └─────────────────┘   └─────────────────┘

    ┌─────────────────────────────────────────────────────────────────┐
    │                    Global Control Plane                         │
    │  • ArgoCD (fleet management across all cells)                  │
    │  • Central observability (Datadog / Grafana Cloud)             │
    │  • Global Kafka (cross-cell event replication via MirrorMaker) │
    │  • ML training pipeline (centralized, reads from all cells)    │
    └─────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions at Each Tier

**10K req/s (P1):**
- CDN offloads ~90% of video download bandwidth
- Redis caching eliminates most MySQL reads
- Rate limiting protects services from abuse
- Still single cluster — walls haven't been hit yet

**100K req/s (P2):**
- Split into serving + data clusters (2 EKS clusters)
- Karpenter replaces managed node groups (faster scaling)
- KEDA for event-driven autoscaling of Kafka consumers
- Pre-computed recommendations with cache (LLM too slow per-request)
- Service mesh for cross-cluster mTLS + traffic shaping

**1M req/s (P3):**
- Cell-based: 3+ regions, each self-contained
- Global routing via Route53 + Global Accelerator
- Cross-cell Kafka replication (MirrorMaker)
- Central ML training, per-cell model serving
- Centralized observability aggregation

### Recommendations Service at Scale

The LLM-per-request model breaks above 10K req/s:

| Scale | Approach |
|-------|----------|
| 1K req/s | LLM call per request (current, 1-10s latency) |
| 10K req/s | Redis cache recommendations (same user = cached for 5-15 min) |
| 100K req/s | Pre-compute top-N per user segment offline, serve from cache |
| 1M req/s | A/B test multiple models, feature flags, real-time ML serving |

### New Components Required Per Tier

| Component | 10K (P1) | 100K (P2) | 1M (P3) |
|-----------|----------|-----------|---------|
| Redis (ElastiCache) | Required | Required | Per-cell |
| CDN (CloudFront) | Required | Required | Per-cell + Global |
| API Gateway | Middleware | Kong/Envoy | Per-cell |
| Service Mesh | — | Istio/Linkerd | Per-cell |
| Karpenter | — | Required | Per-cell |
| KEDA | — | Required | Per-cell |
| Global LB | — | — | Route53 + Global Accelerator |
| Feature Flags | — | — | LaunchDarkly |

---

## Full Roadmap

```
P0 — 1K req/s ✅ DONE
├── HPA for metadata-service, data-service, recommendation-api
├── pgvector connection pooling in recommendation tools
└── AWS managed OpenSearch (replaces self-hosted ES)

P1 — 10K req/s (current priority)
├── Add Redis caching for metadata reads
├── Add API rate limiting middleware
├── Add CloudFront CDN for video downloads
└── Increase MySQL max connections

P2 — 100K req/s
├── Split into serving + data EKS clusters
├── Karpenter for node auto-provisioning
├── KEDA for Kafka consumer autoscaling
├── Service mesh (Istio) for cross-cluster communication
├── Pre-computed recommendations with Redis cache
├── Scale Kafka to 3+ brokers, RF=3 (MSK)
├── Scale OpenSearch to 3+ node cluster
└── PodDisruptionBudgets for all services

P3 — 1M req/s
├── Cell-based architecture (3+ AWS regions)
├── Global routing (Route53 + Global Accelerator)
├── Cross-cell Kafka replication (MirrorMaker)
├── Central ML training + per-cell serving
├── Centralized observability (Grafana Cloud / Datadog)
├── pgvector read replicas or managed RDS Postgres per cell
└── Feature flags for progressive rollouts
```
