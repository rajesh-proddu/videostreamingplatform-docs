# Kubernetes Architecture — Diagrams

Rendered companions to [`KUBERNETES-ARCHITECTURE.md`](./KUBERNETES-ARCHITECTURE.md).

Each diagram is provided as a **PNG** (primary, always renders) with the **Mermaid source**
in a collapsible block. Editable source also lives beside each image under
[`diagrams/`](./diagrams/) (`*.mmd`) — regenerate PNGs with:

```
npx @mermaid-js/mermaid-cli -i diagrams/<name>.mmd -o diagrams/<name>.png -w 1600 -s 2 -b white
```

---

## 1. AWS network topology (VPC · subnets · IGW · NAT · egress)

![AWS network topology](./diagrams/01-network-topology.png)

**Reads:** NLBs + NAT GWs live in public subnets; nodes/RDS/Redis are private-only.
Egress is node &rarr; **per-AZ NAT** &rarr; IGW (dev 2 NAT GWs, prod 3). Video bytes never
transit the cluster — viewers hit CloudFront, which reads the private S3 bucket via OAI.

<details><summary>Mermaid source</summary>

```mermaid
flowchart TB
    internet(("Internet"))
    viewers(("Video viewers"))
    cf["CloudFront CDN<br/>OAI &rarr; S3 (private)"]
    ghcr["ghcr.io / AWS APIs<br/>Bedrock / Razorpay"]

    viewers -->|HTTPS| cf
    cf -->|OAI read| s3

    subgraph VPC["VPC 10.0.0.0/16 &nbsp;(DNS on)"]
        igw["Internet Gateway"]
        nlb["NLB per Service<br/>metadata/data/user/notif"]

        subgraph PUB["PUBLIC subnets &nbsp;(map_public_ip)"]
            direction LR
            nat1["NAT GW AZ-a (+EIP)"]
            nat2["NAT GW AZ-b (+EIP)"]
            nat3["NAT GW AZ-c (+EIP)<br/>prod only"]
        end

        subgraph PRIV["PRIVATE subnets &nbsp;(no public IP)"]
            nodes["EKS worker nodes<br/>(managed node group)<br/>all pods + StatefulSets"]
            rds[("RDS MySQL<br/>db subnet group")]
            redis[("ElastiCache Redis<br/>subnet group")]
        end
    end

    internet <--> igw
    igw <--> nlb
    nlb -->|TCP| nodes
    igw --- nat1 & nat2 & nat3
    nodes -->|"0.0.0.0/0 (per-AZ RT)"| nat1
    nat1 & nat2 & nat3 -->|egress| igw
    igw -.-> ghcr
    nodes --> rds
    nodes --> redis
    s3[("S3 videos bucket<br/>SSE-KMS, versioned")]

    classDef mgd fill:#eef,stroke:#557;
    classDef pub fill:#efe,stroke:#575;
    class rds,redis,s3,cf mgd;
    class nat1,nat2,nat3,nlb pub;
```

</details>

---

## 2. In-cluster namespaces, workloads & data flow

![Workloads and data flow](./diagrams/02-workloads-dataflow.png)

**Reads:** Deployments are stateless (pink = StatefulSets). Kafka is the spine —
`video/watch/subscription-events` fan out to analytics, notifications and search.
`recommendation-api` is ClusterIP-only, reached externally only via the metadata-service proxy.

<details><summary>Mermaid source</summary>

```mermaid
flowchart LR
    subgraph vsp["ns: videostreamingplatform"]
        meta["metadata-service<br/>Deploy x2 · HPA 2-10 · :8080"]
        data["data-service<br/>Deploy x2 · HPA 2-10 · :8081"]
        user["user-service<br/>Deploy x1 · :8082"]
        cdninv["cdn-invalidator<br/>Deploy x1"]
        notif["notifications<br/>Deploy · :8083 · WS gateway"]
        es[("elasticsearch<br/>STS x1 · :9200")]
    end

    subgraph infra["ns: infra"]
        kafka[("kafka<br/>STS x3 KRaft · :9092")]
        pg[("pgvector<br/>STS x1 · :5432")]
    end

    subgraph an["ns: analytics"]
        esc["kafka-es-consumer"]
        whc["watch-history-consumer"]
    end

    subgraph reco["ns: recommendations"]
        rapi["recommendation-api<br/>Deploy x2 · HPA 2-6 · :8000"]
    end

    meta -->|"video-events"| kafka
    data -->|"watch-events"| kafka
    user -->|"subscription-events"| kafka
    kafka -->|video-events| esc --> es
    kafka -->|watch-events| whc -->|Iceberg| glue[("Glue + S3")]
    kafka -->|subscription-events| notif -->|notification-events| kafka
    meta -->|"/recommendations proxy"| rapi
    rapi --> es
    rapi --> pg
    cdninv -->|video-events| kafka
    notif -.->|weekly-report CronJob| pg

    classDef sts fill:#fde,stroke:#a37;
    class kafka,pg,es sts;
```

</details>

---

## 3. Ingress / egress & DNS boundaries

![Ingress egress and DNS](./diagrams/03-ingress-egress-dns.png)

**Reads:** No Ingress controller exists — L4 only (NLB / NodePort). Internal services are
reachable solely by stable cluster DNS, further fenced by NetworkPolicies. All outbound
egress leaves via the per-AZ NAT path.

<details><summary>Mermaid source</summary>

```mermaid
flowchart TB
    ext(("External clients"))
    subgraph cluster["Kubernetes cluster"]
        subgraph edge["External-facing (NLB on AWS / NodePort local)"]
            svcs["metadata :8080 · data :8081<br/>user :8082 · notif :8083"]
        end
        subgraph internal["ClusterIP + headless (internal DNS only)"]
            dns["kafka.infra:9092 (headless)<br/>pgvector.infra:5432<br/>elasticsearch.vsp:9200<br/>recommendation-api:8000"]
        end
        np["NetworkPolicies:<br/>data plane admits only<br/>part-of=videostreamingplatform"]
    end
    egress(("NAT &rarr; IGW &rarr; Internet"))

    ext -->|"L4 LB / NodePort"| svcs
    svcs --> internal
    internal --- np
    cluster -->|"image pulls, Bedrock,<br/>Razorpay, AWS APIs"| egress

    classDef sec fill:#ffe,stroke:#aa5;
    class np sec;
```

</details>

---

## 4. Storage & resiliency (failure blast radius)

![Storage and resiliency](./diagrams/04-storage-resiliency.png)

**Reads:** Kafka (RF=3) and prod RDS (multi-AZ) are the only genuinely HA stores. pgvector
and Elasticsearch run single-replica by design; because EBS volumes are AZ-bound
(`WaitForFirstConsumer`), they recover only if the pod reschedules back into the same AZ.

<details><summary>Mermaid source</summary>

```mermaid
flowchart TB
    subgraph ha["Highly available"]
        kafka["Kafka STS x3<br/>RF=3 · minISR=2<br/>1-broker-per-node anti-affinity"]
        rdsp["RDS (prod)<br/>multi-AZ standby"]
        s3["S3<br/>versioned · SSE-KMS · 11 nines"]
    end
    subgraph spof["Single points of failure (cost tradeoff)"]
        pg["pgvector STS x1"]
        es["elasticsearch STS x1"]
        rdsd["RDS (dev) single-AZ"]
    end
    note["EBS gp3 · RWO · WaitForFirstConsumer<br/>&rarr; PV pinned to one AZ:<br/>reschedule recovers only in the same AZ"]

    pg -.-> note
    es -.-> note
    kafka -.->|survives 1 node/broker loss| ok(("stays writable"))
    pg -.->|node/AZ loss| down(("unavailable until reattach"))
    es -.->|node/AZ loss| down

    classDef good fill:#efe,stroke:#575;
    classDef bad fill:#fee,stroke:#a55;
    class kafka,rdsp,s3 good;
    class pg,es,rdsd bad;
```

</details>
