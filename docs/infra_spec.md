# Infrastructure Specification: K8s Sentinel

## Overview
This document details the infrastructure requirements and Kubernetes manifests needed to deploy the K8s Sentinel Agent and its supporting subsystems (RAG & Event History).

## 1. Core Agent (`sentinel-agent`)
The main brain interfacing with users and the cluster.

### Deployment Strategy
- **Namespace**: `agents` (recommended, based on existing `agents` or `default`).
- **Replicas**: 1
- **Image**: `sentinel-agent:latest` (Base: Python 3.11-slim + `kubectl` + `kcat`).
- **Resources**:
    - Requests: `cpu: 250m`, `memory: 512Mi`
    - Limits: `cpu: 1000m`, `memory: 1Gi`

### Networking
- **Service**: `ClusterIP`
    - Port: 8000 (API)
- **Ingress/Route**: Internal only (accessed via OpenWebUI service-to-service or Ingress if external needed).

### Security (RBAC)
Requires a `ServiceAccount` bound to a `ClusterRole` with:
- `core`: `pods`, `nodes`, `namespaces`, `services`, `events`, `configmaps` (get, list, watch).
- `core`: `pods/log` (get).
- `apps`: `deployments`, `statefulsets`, `daemonsets` (get, list, watch).
- `batch`: `jobs`, `cronjobs` (get, list, watch).
- `metrics.k8s.io`: `pods`, `nodes` (get, list).

---

## 2. Event Storage & History

### Database (Postgres)
We need a Postgres instance for two purposes:
1. **Vector Store**: For RAG embeddings (requires `pgvector`).
2. **Event History**: For storing structured K8s events.

**Proposal**:
- Deploy a dedicated PostgreSQL 16+ instance with `pgvector` enabled in the `agents` namespace OR use an existing infra DB.
- **Storage**: 10Gi PVC.

### Message Broker (Kafka)
- **Current Status**: Existing `my-cluster` in `kafka` namespace.
- **Topic**: `k8s-cluster-events`
    - Partitions: 3
    - Retention: 7 days.

### Event Pipeline (`event-historian`)
A decoupled microservice to handle high-throughput event logging.
- **Deployment**:
    - Sidecar pattern or two small deployments.
    - **Watcher**: Streams K8s events -> Kafka.
    - **Archiver**: Kafka -> Postgres.
- **Dependencies**: Kafka credentials, Postgres connection string.

---

## 3. Knowledge Base Refresh (`knowledge-refresher`)

### CronJob
- **Schedule**: Daily (`0 0 * * *`).
- **Job**:
    - Clones configured git repositories.
    - Processes documents (Markdown, Yaml).
    - Generates embeddings (requires connection to Embedding Model).
    - Updates Postgres Vector Store.
- **Volumes**: Ephemeral `emptyDir` for git cloning.

---

## 4. Configuration Management

**ConfigMap** (`sentinel-config`):
- `LLM_BASE_URL`: URL of the local Qwen model.
- `REDIS_SENTINEL_HOST`: `redis-sentinel.redis-sentinel.svc.cluster.local`.
- `ELASTIC_HOST`: `elasticsearch-es-http.logging.svc.cluster.local`.
- `KAFKA_BOOTSTRAP`: `my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092`.

**Secret** (`sentinel-secrets`):
- `DB_PASSWORD`: Postgres password.
- `GIT_TOKEN`: GitHub/GitLab token for cloning private repos.
