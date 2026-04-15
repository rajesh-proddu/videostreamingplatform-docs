# Scale Analysis & Roadmap

## PRD Target

~1K requests/sec, availability over consistency.

## Current Scale Capacity

**Estimated ceiling: ~200–500 req/s sustained.**

### Component-Level Limits

| Component | Current Config | Ceiling | Bottleneck |
|-----------|---------------|---------|------------|
| metadata-service | 2 replicas, 1 core limit each | ~2K–5K req/s (CRUD) | RDS burst credits, 25-conn pool |
| data-service | 2 replicas, 1 core limit each | ~200–400 concurrent streams | Pod CPU (checksumming), no CDN |
| recommendations | 2 replicas, 1 core limit each | ~2–10 concurrent req | LLM latency (1–10s/call), no pgvector connection pool |
| Kafka | 1 broker, RF=1 | ~50K msg/s | Broker failure = total data loss |
| Elasticsearch | Single node, 256MB heap | ~100 doc/s indexing | Heap exhaustion, single node |
| MySQL (RDS) | db.t3.medium, 25 max conns | ~500–1K queries/s | Burstable CPU, no read replicas |
| pgvector | Single pod, no conn pool in tools | ~100 concurrent queries | New connection per request |

### Key Gaps

- **No HPA** — replica counts are static, no auto-scaling under load — **FIXED in P0**
- **No connection pooling** in recommendation tools (`user_history.py`, `trending.py` create a new `asyncpg.connect()` per call) — **FIXED in P0**
- **No caching layer** (Redis/Memcached) — every metadata read hits MySQL
- **No API gateway** — no rate limiting, no DDoS protection
- **No CDN** — video downloads always go through data-service
- **No circuit breaker** — downstream failures cascade
- **Kafka RF=1** — no fault tolerance
- **Elasticsearch single node, 256MB heap** — undersized for production

---

## Roadmap

### P0 — Required for 1K req/s (DONE)

| Task | Repo | Status |
|------|------|--------|
| Add HPA for metadata-service, data-service | videostreamingplatform | Done |
| Add HPA for recommendation-api | videostreamingplatform-recommendations | Done |
| Fix pgvector connection pooling in recommendation tools | videostreamingplatform-recommendations | Done |
| Use AWS OpenSearch (managed) instead of self-hosted ES in production | videostreamingplatform | Done |

### P1 — Production Hardening

| Task | Repo | Status |
|------|------|--------|
| Add Redis for metadata read caching (video metadata is read-heavy) | videostreamingplatform | Planned |
| Upgrade RDS to non-burstable instance (db.r6g.medium) + read replica | videostreamingplatform | Planned |
| Add API rate limiting (middleware or API gateway) | videostreamingplatform | Planned |
| Increase MySQL max connections to 50+ with read replica routing | videostreamingplatform | Planned |

### P2 — Reliability & Parallelism

| Task | Repo | Status |
|------|------|--------|
| Scale Kafka to 3 brokers, RF=3 | videostreamingplatform-infra | Planned |
| Scale kafka-es-consumer to 3 replicas (match partition count) | videostreamingplatform-analytics | Planned |
| Scale watch-history-consumer to 3+ replicas | videostreamingplatform-analytics | Planned |
| Add PodDisruptionBudgets for all services | All repos | Planned |
| Add liveness/readiness probes to analytics consumers | videostreamingplatform-analytics | Planned |

### P3 — Beyond 1K req/s (5K–10K)

| Task | Repo | Status |
|------|------|--------|
| Add CDN (CloudFront) for video downloads | videostreamingplatform | Planned |
| Add circuit breaker for recommendation service calls | videostreamingplatform | Planned |
| API gateway (Kong/AWS API Gateway) in front of all services | videostreamingplatform-infra | Planned |
| ES/OpenSearch cluster (3+ nodes) | videostreamingplatform | Planned |
| Cross-AZ Kafka deployment | videostreamingplatform-infra | Planned |
| pgvector read replicas or managed RDS Postgres | videostreamingplatform-infra | Planned |
| Hybrid cloud (control plane AWS, data planes multi-cloud) | videostreamingplatform-infra | Planned |
