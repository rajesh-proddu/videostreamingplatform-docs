# Video Streaming Platform — Documentation

Central documentation for the video streaming platform.

## Contents

- **[ARCHITECTURE.md](ARCHITECTURE.md)** — System architecture, service details, data flow, event schemas, deployment topology, and technology stack

## Related Repositories

| Repository | Description |
|-----------|-------------|
| [videostreamingplatform](https://github.com/rajesh-proddu/videostreamingplatform) | Core platform — Metadata Service + Data Service (Go) |
| [videostreamingplatform-analytics](https://github.com/rajesh-proddu/videostreamingplatform-analytics) | Data pipelines — Kafka→ES, Spark→Iceberg (Python) |
| [videostreamingplatform-recommendations](https://github.com/rajesh-proddu/videostreamingplatform-recommendations) | AI recommendation engine — LangGraph agent (Python) |
| [videostreamingplatform-schemas](https://github.com/rajesh-proddu/videostreamingplatform-schemas) | Event schemas — Avro + Protobuf definitions |
| [videostreamingplatform-infra](https://github.com/rajesh-proddu/videostreamingplatform-infra) | Shared infrastructure — Kafka, pgvector, Glue, ArgoCD |
