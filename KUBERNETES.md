# Kubernetes Role in the Video Streaming Platform

This document describes how Kubernetes is used across the entire video streaming platform — from local development on a Kind cluster to production on AWS EKS.

---

## 1. Cluster Topology

The platform runs on a **single Kubernetes cluster** with workloads segregated by namespace. Every namespace is created by the infra repo (`videostreamingplatform-infra/networking/namespaces.yaml`) and labeled with `app.kubernetes.io/part-of: videostreamingplatform` for network policy targeting.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                              │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ argocd namespace                                                 │  │
│  │  └─ ArgoCD controller (watches all repos, auto-syncs)           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ infra namespace                                 (infra repo)     │  │
│  │  ├─ Kafka StatefulSet (KRaft, 1 replica)                        │  │
│  │  ├─ Kafka headless Service (:9092, :9093)                       │  │
│  │  ├─ kafka-topics-init Job (creates video-events, watch-events)  │  │
│  │  ├─ pgvector StatefulSet (Postgres 16 + vector ext, 1 replica)  │  │
│  │  ├─ pgvector Service (:5432)                                    │  │
│  │  ├─ pgvector-secret Secret                                      │  │
│  │  ├─ LocalStack Deployment (Glue schema registry)                │  │
│  │  └─ LocalStack Service (:4566)                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ videostreamingplatform namespace              (main repo)        │  │
│  │  ├─ metadata-service Deployment (Go, :8080)                     │  │
│  │  ├─ data-service Deployment (Go, :8081 HTTP, :50051 gRPC)       │  │
│  │  ├─ MySQL Deployment + PVC (videoplatform DB)                   │  │
│  │  ├─ mysql-init ConfigMap (DDL for videos, uploads, downloads)   │  │
│  │  ├─ MinIO Deployment + bucket-init Job (S3-compatible storage)  │  │
│  │  ├─ Elasticsearch Deployment (single-node, :9200)               │  │
│  │  ├─ app-config ConfigMap (all service env vars)                 │  │
│  │  ├─ db-credentials Secret                                       │  │
│  │  ├─ metadata-service NodePort Service (:30080 → :8080)          │  │
│  │  └─ data-service NodePort Service (:30081 → :8081)              │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ analytics namespace                          (analytics repo)    │  │
│  │  ├─ kafka-es-consumer Deployment (Python, 1 replica)            │  │
│  │  ├─ watch-history-consumer Deployment (Python, 1 replica)       │  │
│  │  ├─ catalog-init Job (creates Iceberg tables)                   │  │
│  │  └─ iceberg-compaction CronJob (weekly, Sundays 03:00)          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ recommendations namespace              (recommendations repo)    │  │
│  │  ├─ recommendation-api Deployment (FastAPI, 2 replicas)         │  │
│  │  ├─ recommendation-api Service (:8000)                          │  │
│  │  ├─ recommendation-config ConfigMap                              │  │
│  │  └─ pgvector-credentials Secret                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ observability namespace                       (main repo)        │  │
│  │  ├─ Jaeger Deployment (with Badger persistent storage)          │  │
│  │  ├─ Prometheus Deployment                                        │  │
│  │  └─ Grafana Deployment + dashboards ConfigMaps                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Local vs AWS Namespace Differences

| Concern | Local (Kind) | AWS (EKS) |
|---------|-------------|-----------|
| Observability location | Separate `observability` namespace | Separate `observability` namespace |
| MySQL | In-cluster Deployment | AWS RDS (out-of-cluster) |
| S3 | MinIO in-cluster | AWS S3 (out-of-cluster) |
| Service type | NodePort (forwarded by Kind) | LoadBalancer (NLB) |
| Replicas (services) | 1 | 2 |
| Security context | Not set | `readOnlyRootFilesystem`, `runAsNonRoot`, `runAsUser: 1000` |

---

## 2. Workload Types

Kubernetes is used for five distinct workload types across the platform:

### 2a. Long-Running Services (Deployments)

| Deployment | Namespace | Replicas | Strategy | Probes |
|-----------|-----------|----------|----------|--------|
| `metadata-service` | videostreamingplatform | 1 (local) / 2 (AWS) | RollingUpdate (maxSurge=1, maxUnavailable=0) | liveness + readiness on `/health` |
| `data-service` | videostreamingplatform | 1 (local) / 2 (AWS) | RollingUpdate (maxSurge=1, maxUnavailable=0) | liveness + readiness on `/health` |
| `recommendation-api` | recommendations | 2 | Default | liveness `/health`, readiness `/ready` |
| `kafka-es-consumer` | analytics | 1 | Default | None |
| `watch-history-consumer` | analytics | 1 | Default | None |
| `mysql` | videostreamingplatform | 1 | Default | liveness + readiness via `mysqladmin ping` |
| `minio` | videostreamingplatform | 1 | Default | liveness + readiness on `/minio/health/*` |
| `elasticsearch` | videostreamingplatform | 1 | Default | readiness on `/_cluster/health` |
| `jaeger` | videostreamingplatform / observability | 1 | Default | liveness + readiness on UI port |
| `prometheus` | videostreamingplatform / observability | 1 | Default | readiness on `/-/ready` |
| `grafana` | videostreamingplatform / observability | 1 | Default | readiness on `/api/health` |
| `localstack` | infra | 1 | Default | readiness on `/_localstack/health` |

### 2b. Stateful Services (StatefulSets)

| StatefulSet | Namespace | Volume | Storage |
|------------|-----------|--------|---------|
| `kafka` | infra | `kafka-data` PVC | 5Gi |
| `pgvector` | infra | `pgvector-data` PVC | 5Gi |

Both use headless Services (`clusterIP: None`) for stable DNS names. Kafka uses a headless Service because KRaft controller quorum voters reference the pod-specific DNS (`kafka-0.kafka.infra.svc.cluster.local:9093`).

### 2c. One-Shot Init Jobs

| Job | Namespace | Purpose | Runs |
|-----|-----------|---------|------|
| `kafka-topics-init` | infra | Creates `video-events` (3p) and `watch-events` (6p) topics | On first deploy |
| `minio-create-bucket` | videostreamingplatform | Creates `video-platform-storage` S3 bucket | On first deploy |
| `catalog-init` | analytics | Creates Iceberg `analytics.watch_history` table via Glue | On first deploy |

All init Jobs include an **init container** that waits for the dependency to be healthy before running:
- `kafka-topics-init` → waits for `kafka:9092` via `nc -z`
- `minio-create-bucket` → waits for MinIO `/minio/health/ready` via `wget`
- `catalog-init` → depends on Glue being available

### 2d. Scheduled Jobs (CronJobs)

| CronJob | Namespace | Schedule | Purpose |
|---------|-----------|----------|---------|
| `iceberg-compaction` | analytics | `0 3 * * 0` (Sundays 03:00) | Rewrites small Iceberg Parquet files into larger ones |

### 2e. Database Schema Initialization (ConfigMap)

The `mysql-init` ConfigMap in the `videostreamingplatform` namespace contains DDL that runs on MySQL first boot:
- `videos` table (id, title, description, duration, size_bytes, upload_status, timestamps)
- `uploads` table (id, video_id, user_id, chunk tracking, status — FK to videos)
- `downloads` table (id, video_id, user_id, status — FK to videos)
- `users` table (id, username)

Mounted at `/docker-entrypoint-initdb.d` in the MySQL container so it runs automatically on first start.

---

## 3. Networking

### 3a. Service Discovery

All inter-service communication uses Kubernetes DNS. The canonical pattern is:

```
<service>.<namespace>.svc.cluster.local:<port>
```

Key cross-namespace DNS entries used in ConfigMaps:

| From | To | DNS | Port |
|------|----|-----|------|
| metadata-service, data-service | Kafka | `kafka.infra.svc.cluster.local` | 9092 |
| metadata-service | Recommendation API | `recommendations.recommendations.svc.cluster.local` | 8000 |
| kafka-es-consumer, watch-history-consumer | Kafka | `kafka.infra.svc.cluster.local` | 9092 |
| kafka-es-consumer, recommendations | Elasticsearch | `elasticsearch.videostreamingplatform.svc.cluster.local` | 9200 |
| recommendations | pgvector | `pgvector.infra.svc.cluster.local` | 5432 |
| metadata-service, data-service | Jaeger | `jaeger.videostreamingplatform.svc.cluster.local` | 4318 |
| metadata-service, data-service | MySQL | `mysql.videostreamingplatform.svc.cluster.local` | 3306 |
| data-service | MinIO | `minio.videostreamingplatform.svc.cluster.local` | 9000 |

### 3b. External Access

**Local (Kind)**: NodePort Services are used with Kind's `extra_port_mappings` to forward from the host machine:

| Service | NodePort | Host Port |
|---------|----------|-----------|
| metadata-service | 30080 | 8080 |
| data-service | 30081 | 8081 |
| Prometheus | 30090 | 9090 |
| Jaeger UI | 30686 | 16686 |
| Grafana | 30300 | 3000 |

**AWS (EKS)**: LoadBalancer Services with AWS NLB annotations:

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
```

Both `metadata-service` and `data-service` get their own NLB endpoints.

### 3c. Network Policies

Defined in `videostreamingplatform-infra/networking/network-policies.yaml`, all applied to the `infra` namespace to control ingress to shared infrastructure:

| Policy | Target Pod | Allowed Sources | Port |
|--------|-----------|----------------|------|
| `allow-kafka-access` | `app: kafka` | Any namespace with label `app.kubernetes.io/part-of: videostreamingplatform` | 9092/TCP |
| `allow-pgvector-access` | `app: pgvector` | Same label selector | 5432/TCP |
| `allow-localstack-access` | `app: localstack` | Same label selector | 4566/TCP |

The namespace label `app.kubernetes.io/part-of: videostreamingplatform` is applied to `infra`, `videostreamingplatform`, `analytics`, and `recommendations` — meaning all four namespaces can reach Kafka, pgvector, and LocalStack. Finer-grained restrictions (e.g. only recommendations can write to pgvector) are not enforced at the K8s level.

---

## 4. Configuration & Secrets

### 4a. ConfigMaps

Each namespace has its own ConfigMap with environment-specific values:

| ConfigMap | Namespace | Key Contents |
|-----------|-----------|-------------|
| `app-config` | videostreamingplatform | MySQL host/port/user/db, S3 bucket/region/endpoint, OTEL endpoint, Kafka brokers/topics, upload store mode, recommendation service URL |
| `recommendation-config` | recommendations | LLM provider, Bedrock model ID, pgvector URL, ES URL, Kafka brokers, API port |
| `prometheus-config` | videostreamingplatform | Prometheus scrape config (scrapes metadata-service:8080 and data-service:8081) |
| `grafana-datasources` | videostreamingplatform | Prometheus datasource configuration |
| `grafana-dashboard-provider` | videostreamingplatform | Dashboard file provider path |
| `grafana-dashboards` | videostreamingplatform | Created dynamically from `scripts/grafana/dashboards/` by `deploy.sh` |
| `mysql-init` | videostreamingplatform | SQL DDL for initial schema |

The analytics consumers don't use ConfigMaps — their environment variables are inline in the Deployment spec, pointing directly at the `kafka.infra` and `elasticsearch.videostreamingplatform` DNS names.

### 4b. Secrets

| Secret | Namespace | Contents |
|--------|-----------|---------|
| `db-credentials` | videostreamingplatform | MySQL password (mounted as env var `MYSQL_PASSWORD` in metadata-service) |
| `pgvector-secret` | infra | pgvector Postgres password |
| `pgvector-credentials` | recommendations | pgvector DSN for the recommendation service |
| `rds-credentials` (AWS) | videostreamingplatform | RDS host, username, password (created externally, consumed by metadata-service) |

---

## 5. Storage

### 5a. Persistent Volumes

| PVC / VolumeClaimTemplate | Namespace | Size | Used By |
|--------------------------|-----------|------|---------|
| `mysql-pvc` | videostreamingplatform | 1Gi | MySQL Deployment |
| `kafka-data` (VCT) | infra | 5Gi | Kafka StatefulSet |
| `pgvector-data` (VCT) | infra | 5Gi | pgvector StatefulSet |

MinIO, Prometheus, Jaeger (local), and Grafana use `emptyDir` volumes — data is ephemeral and lost on pod restart. In production, the Jaeger AWS manifest uses Badger persistent storage.

### 5b. AWS Managed Storage (Production)

In production, Kubernetes does **not** manage these data stores — Terraform provisions them outside the cluster:

| Resource | Terraform File | Kubernetes Equivalent |
|----------|---------------|----------------------|
| AWS RDS (MySQL 8.0) | `rds.tf` | Replaces in-cluster MySQL |
| AWS S3 bucket | `storage.tf` | Replaces in-cluster MinIO |

Services connect to RDS and S3 via environment variables injected from ConfigMaps/Secrets. The connection is authenticated via IRSA (see section 7).

---

## 6. Local Cluster Provisioning (Kind + Terraform)

The local Kind cluster is provisioned via Terraform (`k8s/local/terraform/kind.tf`):

```
Terraform (tehcyx/kind provider)
  └─ kind_cluster "videostreamingplatform"
       ├─ control-plane node (extra_port_mappings for NodePorts)
       └─ worker node
       └─ Kubernetes v1.32.2
```

The `deploy.sh` script orchestrates the full lifecycle:

```bash
./k8s/local/deploy.sh up       # Create cluster → build images → load into Kind → deploy manifests
./k8s/local/deploy.sh down     # Terraform destroy the cluster
./k8s/local/deploy.sh status   # Show pods, services, and access URLs
./k8s/local/deploy.sh rebuild  # Rebuild Docker images → load into Kind → rollout restart
```

### Deploy Manifest Order (enforced by `deploy.sh`)

1. `namespace.yaml` — Create the `videostreamingplatform` namespace
2. `secrets.yaml` — DB credentials
3. `mysql-init.yaml` — Schema DDL ConfigMap
4. `configmap.yaml` — Application environment config
5. `mysql.yaml` — MySQL Deployment + PVC + Service
6. `minio.yaml` — MinIO Deployment + Service + bucket-init Job
7. `observability.yaml` — Jaeger + Prometheus + Grafana (all in one file)
8. Grafana dashboards ConfigMap (created dynamically from files)
9. `services.yaml` — NodePort Services for metadata-service and data-service
10. `metadata-service-deploy.yaml` — Metadata Service Deployment
11. `data-service-deploy.yaml` — Data Service Deployment

Between steps 6 and 9, the script runs `kubectl wait` for MySQL and MinIO readiness before deploying the application services.

### Image Loading

Kind clusters can't pull from Docker Hub by default for local images. The deploy script builds images locally then loads them with `kind load docker-image`:

```bash
docker build -t videostreamingplatform/metadata-service:latest -f build/docker/metadataservice.Dockerfile .
docker build -t videostreamingplatform/data-service:latest -f build/docker/dataservice.Dockerfile .
kind load docker-image videostreamingplatform/metadata-service:latest --name videostreamingplatform
kind load docker-image videostreamingplatform/data-service:latest --name videostreamingplatform
```

Both Deployments use `imagePullPolicy: IfNotPresent` so they use the pre-loaded images.

---

## 7. AWS EKS Production Cluster (Terraform)

The production cluster is fully provisioned by Terraform (`k8s/aws/terraform/`):

### 7a. Infrastructure Resources

| Terraform File | Resources Created |
|---------------|-------------------|
| `vpc.tf` | VPC, public/private subnets (multi-AZ), Internet Gateway, NAT Gateways, route tables |
| `eks.tf` | EKS cluster, managed node group, OIDC provider, IRSA roles, security groups |
| `rds.tf` | RDS MySQL 8.0 instance, subnet group, security group, KMS encryption key |
| `storage.tf` | S3 bucket (versioned, KMS-encrypted, lifecycle rules, public access blocked) |

### 7b. EKS Cluster Configuration

| Parameter | Dev | Prod |
|-----------|-----|------|
| Kubernetes version | 1.29 | 1.32 |
| Node instance type | t3.small | t3.medium |
| Node group: min/desired/max | 1/2/3 | 3/5/10 |
| Node AMI | AL2023_x86_64_STANDARD | AL2023_x86_64_STANDARD |
| Subnets | 2 public + 2 private | 3 public + 3 private |
| EKS logs | api, audit, authenticator, controllerManager, scheduler | Same |

Worker nodes run in **private subnets** only. The EKS API endpoint is both public and private.

### 7c. IRSA (IAM Roles for Service Accounts)

IRSA allows Kubernetes ServiceAccounts to assume AWS IAM roles without static credentials. The EKS OIDC provider is registered, then trust policies bind specific ServiceAccounts to IAM roles:

| Service | ServiceAccount | IAM Role | Permissions |
|---------|---------------|----------|-------------|
| `metadata-service` | `metadata-service` (videostreamingplatform ns) | `metadata-service-irsa-{env}` | `rds-db:connect` to the RDS instance |
| `data-service` | `data-service` (videostreamingplatform ns) | `data-service-irsa-{env}` | S3 Get/Put/Delete/List on the videos bucket + KMS Decrypt/GenerateDataKey |

The AWS K8s manifests include `ServiceAccount` resources annotated with `eks.amazonaws.com/role-arn`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: data-service
  namespace: videostreamingplatform
  annotations:
    eks.amazonaws.com/role-arn: ${IRSA_DATA_ROLE_ARN}
```

### 7d. RDS Configuration

| Parameter | Dev | Prod |
|-----------|-----|------|
| Instance class | db.t3.micro | db.t3.medium |
| Storage | 20GB gp2 (auto-scale to 100GB) | Same |
| Multi-AZ | No | Yes |
| Backup retention | 1 day | 7 days |
| Final snapshot | Skipped | Required |
| Deletion protection | No | Yes |
| Encryption | KMS (key rotation enabled) | Same |

RDS lives in private subnets. Its security group only allows port 3306 from the EKS node security group and the EKS cluster security group.

### 7e. S3 Configuration

- Bucket name: `videostreamingplatform-videos-{env}-{account_id}`
- Versioning: Enabled
- Encryption: KMS with key rotation
- Lifecycle: Videos expire after 30 days (dev) or 365 days (prod); non-current versions after 30 days
- Public access: Fully blocked (all four settings)

### 7f. AWS Deployment Differences

The AWS K8s manifests in `k8s/aws/manifests/` differ from local in several ways:

| Aspect | Local | AWS |
|--------|-------|-----|
| Service type | NodePort | LoadBalancer (NLB) |
| Replicas | 1 | 2 |
| Image source | `videostreamingplatform/*:latest` (loaded into Kind) | `ghcr.io/rajesh-proddu/videostreamingplatform/*:latest` |
| Image pull policy | IfNotPresent | Always |
| MySQL source | In-cluster Deployment | RDS (host from Secret) |
| S3 source | In-cluster MinIO | AWS S3 (bucket from ConfigMap) |
| OTEL endpoint | `jaeger.videostreamingplatform.svc:4318` | `jaeger.observability.svc:4318` |
| Security context | None | readOnlyRootFilesystem, runAsNonRoot, runAsUser 1000 |
| Pod anti-affinity | None | Preferred anti-affinity on `kubernetes.io/hostname` (spreads replicas across nodes) |
| Auth to AWS services | Static credentials in ConfigMap | IRSA (zero static credentials) |

---

## 8. GitOps with ArgoCD

ArgoCD runs in the `argocd` namespace and provides continuous deployment from Git. It is configured via manifests in `videostreamingplatform-infra/argocd/`.

### 8a. AppProjects

Two projects control which repos can deploy to which namespaces:

| Project | Source Repos | Target Namespaces |
|---------|-------------|-------------------|
| `platform` | videostreamingplatform, videostreamingplatform-infra | `videostreamingplatform`, `infra` |
| `data` | videostreamingplatform-analytics, videostreamingplatform-recommendations | `analytics`, `recommendations` |

### 8b. Applications

| Application | Source Repo | Source Path | Target Namespace | Branch |
|------------|------------|-------------|-----------------|--------|
| `videostreamingplatform` | videostreamingplatform | `k8s/local` | videostreamingplatform | `main` |
| `infra` | videostreamingplatform-infra | (root) | infra | `main` |
| `analytics` | videostreamingplatform-analytics | `k8s` | analytics | `main` |
| `recommendations` | videostreamingplatform-recommendations | `k8s` | recommendations | `main` |

All four Applications have the same sync policy:

```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources removed from Git
    selfHeal: true   # Revert manual cluster changes
  syncOptions:
    - CreateNamespace=true
```

### 8c. Deployment Flow

```
Developer pushes to main/master
  → GitHub Actions CI runs (tests, lint, build)
  → Docker image pushed to ghcr.io (build.yml workflow)
  → ArgoCD detects repo change (polling or webhook)
  → ArgoCD syncs K8s manifests from the repo's k8s/ directory
  → Kubernetes performs RollingUpdate
  → New pods pass readiness probes → old pods terminate
```

---

## 9. Health Checks & Rolling Updates

### 9a. Probe Configuration

All application services follow the same probe pattern:

| Probe | Path | Initial Delay | Period | Timeout | Failure Threshold |
|-------|------|--------------|--------|---------|-------------------|
| Liveness | `/health` | 15s (local) / 30s (AWS) | 10s | 5s | 3 |
| Readiness | `/health` (or `/ready`) | 5s (local) / 10s (AWS) | 5s | 2s | 2 |

The distinction: liveness failure restarts the pod; readiness failure removes it from the Service endpoints (stops receiving traffic) but doesn't restart it.

### 9b. Rolling Update Strategy

Both Go services use:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Create 1 new pod before terminating old
    maxUnavailable: 0  # Never have fewer than desired replicas
```

This means zero-downtime deployments: a new pod must pass readiness before the old pod is terminated.

### 9c. Pod Anti-Affinity (AWS only)

The AWS manifests include preferred pod anti-affinity to spread replicas across nodes:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: [data-service]
          topologyKey: kubernetes.io/hostname
```

This is "preferred" not "required" — if there aren't enough nodes, replicas can colocate.

---

## 10. Resource Management

### Resource Requests and Limits

| Deployment | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|------------|-----------|----------------|-------------|
| metadata-service | 100m (local) / 200m (AWS) | 500m (local) / 1000m (AWS) | 128Mi (local) / 256Mi (AWS) | 512Mi (local) / 1Gi (AWS) |
| data-service | 200m | 1000m | 256Mi | 1Gi |
| recommendation-api | 200m | 1000m | 256Mi | 512Mi |
| kafka-es-consumer | 100m | 500m | 128Mi | 256Mi |
| watch-history-consumer | 100m | 250m | 256Mi | 512Mi |
| mysql | 100m | 500m | 256Mi | 512Mi |
| minio | 100m | 500m | 128Mi | 512Mi |
| elasticsearch | — | — | 512Mi | 1Gi |
| jaeger | 100m | 500m | 128Mi | 512Mi |
| prometheus | 100m | 500m | 128Mi | 512Mi |
| grafana | 100m | 500m | 128Mi | 512Mi |

---

## 11. CI/CD Pipeline Integration

### GitHub Actions → GHCR → ArgoCD

The `build.yml` workflow in the main repo triggers on pushes to `main` that touch service code:

1. **Test** — runs the `test.yml` reusable workflow
2. **Build** — matrix build of metadata-service and data-service Docker images
3. **Push** — pushes to `ghcr.io/rajesh-proddu/videostreamingplatform/<service>` with tags:
   - `:<commit-sha>` (always)
   - `:latest` (on main branch)
   - `:<version>`, `:<major>.<minor>`, `:<major>` (on `v*` tags)
4. **Scan** — Trivy vulnerability scan, results uploaded as SARIF to GitHub Security tab
5. **Deploy** — ArgoCD picks up the updated `:latest` image and syncs

### Image References

| Service | Local Image | Production Image |
|---------|------------|-----------------|
| metadata-service | `videostreamingplatform/metadata-service:latest` | `ghcr.io/rajesh-proddu/videostreamingplatform/metadata-service:latest` |
| data-service | `videostreamingplatform/data-service:latest` | `ghcr.io/rajesh-proddu/videostreamingplatform/data-service:latest` |
| kafka-es-consumer | built locally | `ghcr.io/rajesh-proddu/videostreamingplatform-analytics/kafka-es-consumer:latest` |
| watch-history-consumer | built locally | `ghcr.io/rajesh-proddu/videostreamingplatform-analytics/watch-history-consumer:latest` |
| catalog-admin | built locally | `ghcr.io/rajesh-proddu/videostreamingplatform-analytics/catalog-admin:latest` |
| recommendation-api | built locally | `ghcr.io/rajesh-proddu/videostreamingplatform-recommendations/recommendations:latest` |

---

## 12. RBAC

Each namespace has its own ServiceAccount defined in `videostreamingplatform-infra/rbac/service-accounts.yaml`:

| ServiceAccount | Namespace |
|---------------|-----------|
| `videostreamingplatform-sa` | videostreamingplatform |
| `analytics-sa` | analytics |
| `recommendations-sa` | recommendations |
| `infra-sa` | infra |

In AWS, additional per-service ServiceAccounts are created with IRSA annotations (see section 7c). The recommendation service's Deployment explicitly uses `serviceAccountName: recommendations-sa`.

---

## 13. Manifest Ownership by Repository

Each repo owns the K8s manifests for its own workloads. ArgoCD watches the `k8s/` directory of each repo:

| Repository | Manifest Path | What It Deploys |
|-----------|---------------|-----------------|
| **videostreamingplatform-infra** | Root (`kafka/`, `pgvector/`, `networking/`, `rbac/`, `schema-registry/`) | Namespaces, Kafka, pgvector, LocalStack, ServiceAccounts, NetworkPolicies |
| **videostreamingplatform** | `k8s/local/` (local) or `k8s/aws/manifests/` (AWS) | Both Go services, MySQL, MinIO, ES, observability stack, ConfigMaps, Secrets |
| **videostreamingplatform-analytics** | `k8s/` | Both Kafka consumers, catalog-init Job, compaction CronJob |
| **videostreamingplatform-recommendations** | `k8s/` | Recommendation API Deployment + Service + ConfigMap + Secret |
| **videostreamingplatform-infra** | `argocd/` | ArgoCD Applications and AppProjects (not synced by ArgoCD itself — applied manually) |
