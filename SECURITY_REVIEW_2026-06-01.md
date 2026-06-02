# Security review — workspace summary (mode: full / entire-repo-code)

Run: `security-review-20260601-1431` · 6 repos in scope, **all 6 reviewed** (core repo re-run after an initial socket-error failure).

## Per-repo results

| Repo | Mode | Files in scope | C | H | M | L | I | Report |
|---|---|---:|---:|---:|---:|---:|---:|---|
| videostreamingplatform (core Go) | full | 91 | 1 | 4 | 2 | 3 | 0 | `videostreamingplatform.md` |
| videostreamingplatform-infra | full | 38 | 0 | 1 | 3 | 3 | 1 | `videostreamingplatform-infra.md` |
| videostreamingplatform-recommendations | full | ~30 | 0 | 0 | 1 | 3 | 2 | `videostreamingplatform-recommendations.md` |
| videostreamingplatform-analytics | full | 23 | 0 | 0 | 0 | 3 | 2 | `videostreamingplatform-analytics.md` |
| videostreamingplatform-schemas | full | 11 | 0 | 0 | 0 | 0 | 1 | `videostreamingplatform-schemas.md` |
| videostreamingplatform-e2e | full | 56 | 0 | 0 | 0 | 0 | 2 | `videostreamingplatform-e2e.md` |

**Cross-repo tally (6 reviewed):** 1 CRITICAL · 5 HIGH · 6 MEDIUM · 12 LOW · 8 INFO

## CRITICAL / HIGH findings (sorted by severity)

### [CRITICAL] CloudFront fronts the videos bucket with no signed-URL auth — full paywall bypass when CDN enabled
- `videostreamingplatform/k8s/aws/terraform/cloudfront.tf:29-77`, `metadataservice/handlers/video.go:47-52`, `dataservice/main.go:116-123`
- **Exploit:** CloudFront serves `/videos/{id}` straight from S3 via OAI with no `trusted_signers`/key groups. metadataservice hands every (unauthenticated) `GET /videos` response a `PlaybackURL = CDN_BASE_URL + /videos/{id}`. When `CDN_BASE_URL` is set, an anonymous client lists videos on the public NLB, reads each id, and fetches the CDN URL directly — JWT + subscription entitlement check (which lives in dataservice) is entirely bypassed because playback no longer transits dataservice. Latent until CDN playback is enabled, but cloudfront.tf provisions it for exactly that purpose.
- **Fix:** Require CloudFront signed URLs/cookies; mint short-lived signed URLs only after the entitlement check, or validate JWT at the edge (Lambda@Edge/CF Functions).

### [HIGH] Mock payment provider shipped as default in the AWS manifest with a hardcoded webhook secret — free subscriptions
- `videostreamingplatform/k8s/aws/manifests/user-service-deploy.yaml:61-82`, `userservice/payment/mock.go:20-25,55-77`, `userservice/handlers/webhook.go:62-99`
- **Exploit:** AWS manifest sets `PAYMENT_PROVIDER: "mock"` and makes the webhook secret optional → `NewMockProvider("")` falls back to the hardcoded `mock_webhook_secret`. Two free-subscription paths: (1) authed user hits the public `GET /mock/checkout?ref=<their-ref>` to self-fire a signed `payment_link.paid`; (2) attacker forges a `payment_link.paid` body HMAC-signed with the public-knowledge `mock_webhook_secret`. Either flips entitlement to `true` on next `/auth/refresh`.
- **Fix:** Never default `PAYMENT_PROVIDER=mock` in cloud; require `razorpay` or fail closed. Gate `/mock/checkout` behind a non-prod flag. Remove the hardcoded secret fallback.

### [HIGH] All dataservice upload endpoints are unauthenticated — anyone can overwrite any video object
- `videostreamingplatform/dataservice/main.go:109-112`, `dataservice/handlers/upload.go:56-94,326-381`
- **Exploit:** `POST /uploads/initiate|/chunks|/complete` are bare `HandleFunc` with no JWT/ownership check. dataservice is a public NLB. Attacker initiates an upload with `video_id` = any known id and completes it, overwriting/defacing the real video served to paying users. The size guard is attacker-controlled (`TotalSize` in same request), so storage abuse is unbounded too.
- **Fix:** Wrap upload routes in `JWTAuth`; enforce uploader owns the `video_id`; reject initiate if the target key exists and caller isn't the owner.

### [HIGH] metadata-service video CRUD is fully unauthenticated
- `videostreamingplatform/metadataservice/main.go:126-131`, `metadataservice/handlers/video.go`
- **Exploit:** `POST/PUT/DELETE /videos[/{id}]` carry only RateLimiter+logging — no auth, no ownership. On a public NLB, any anonymous client can enumerate the catalog, tamper with metadata, or **`DELETE /videos/{id}`**. Combined with open uploads + CDN bypass, the content plane has no authorization at all.
- **Fix:** Require `JWTAuth` on mutating routes; add an ownership/role model; decide explicitly whether listing is public.

### [HIGH] dataservice gRPC server has no authentication or TLS (+ cross-tenant ListUploads IDOR)
- `videostreamingplatform/dataservice/main.go:178-185`, `dataservice/server/grpc_server.go`
- **Exploit:** `grpc.NewServer()` with no interceptor and no TLS (plaintext h2c). Anyone reaching :50051 can drive uploads; `ListUploads` returns arbitrary users' records by request-supplied `user_id` (IDOR). Blast radius currently in-cluster (LB exposes 8081, not 50051) — no defense-in-depth.
- **Fix:** Add a JWT-validating unary interceptor, enable TLS/mTLS, and scope `ListUploads` to the authenticated subject.

### [HIGH] Hardcoded pgvector password (`recopass`) committed and shipped to prod EKS via plaintext ConfigMaps
- `videostreamingplatform-infra/charts/recommendations/values.yaml:29` + `values-aws.yaml:21` (prod overlay), `charts/analytics/values.yaml:103,124`, `pgvector/deployment.yaml:65`, `docker-compose.infra.yml:162`
- **Exploit:** `recouser:recopass` is committed and injected as a plain ConfigMap (not Secret). Anyone with repo read access or `kubectl get configmap -o yaml` gets the prod DB password; on EKS with default VPC CNI the NetworkPolicies are likely unenforced, so any pod cluster-wide can reach `pgvector.infra.svc.cluster.local:5432` and authenticate → full read/write to watch-history feature vectors + trending data.
- **Fix:** Generate the password at bootstrap (mirror `seed_auth_secret`), store in a k8s Secret, reference via `secretKeyRef`; never embed in a ConfigMap or `values*.yaml`. Rotate `recopass`. Confirm NetworkPolicy enforcement is actually on.

## MEDIUM findings (one-liners)

- **[M] userservice fail-open JWT secret** — `userservice/main.go:36-40`. Empty `JWT_SIGNING_SECRET` → signs with hardcoded `dev-insecure-jwt-secret` (and dataservice disables the paywall entirely). Not exploitable as shipped (secret is required in the AWS manifest) but a latent misconfig trap. Fix: fail closed in non-dev.
- **[M] No rate limiting on userservice auth endpoints** — `userservice/main.go:97-123`. `/auth/login` (bcrypt) and `/auth/register` unthrottled on a public NLB → credential brute force + bcrypt-CPU DoS. Fix: apply the existing `RateLimiter` + lockout.
- **[M] NetworkPolicies likely unenforced on EKS** — `infra/networking/network-policies.yaml`. Default VPC CNI doesn't enforce NetworkPolicy unless `ENABLE_NETWORK_POLICY=true`/Calico; nothing enables it. All four policies silently no-op → in-cluster segmentation is a false assumption underpinning the HIGH above and the next two.
- **[M] Elasticsearch runs with security disabled** — `infra/elasticsearch/elasticsearch.yaml:35-36` (`xpack.security.enabled:false`). No auth/TLS on :9200; any pod reaching it gets unauthenticated read/write/delete on the `videos` index.
- **[M] Kafka brokers fully PLAINTEXT, no auth/ACLs** — `infra/kafka/deployment.yaml:70-77`. Any client reaching :9092 can produce/consume any topic → forge `watch-events`/`video-events` (poison analytics + reco embeddings) or read watch-history PII.
- **[M] pgvector creds in plaintext GitOps ConfigMap; Secret bypassed** — `recommendations/k8s/configmap.yaml:10` + `deployment.yaml:27-29`. Deployment wires creds via `configMapRef` only; the `pgvector-credentials` Secret is dead manifest. (Same class as the infra HIGH, capped M because cluster-internal.)

## Repos with no actionable findings
- **schemas** (1 INFO) and **e2e** (2 INFO) are deliberately low-surface; only dev/LocalStack default creds and a safe CI `env:` pattern. Effectively clean.

## Reviewed-and-clean (core repo) — notable
JWT verify (rejects `alg=none`/HS-RS confusion, enforces `typ`), Razorpay webhook raw-body HMAC + idempotency ledger + double-capture guard, SQL parameterization, the metadata→recommendations proxy (no SSRF; URL is server-config, user input goes in the body), and S3/RDS/IRSA Terraform (SSE-KMS, no public access, least-privilege IRSA) were all reviewed and found sound.
