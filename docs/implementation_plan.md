# Implementation Plan - K8s Sentinel Agent

This agent will act as a sentinel for a Kubernetes cluster, answering natural language questions about the cluster's state using a local Qwen3-4b model. It will be built with LangGraph and exposed via an API compatible with OpenWebUI.

## User Review Required
> [!IMPORTANT]
> **LLM Endpoint**: Integration with `qwen-agent` or `vllm` (Configurable endpoint).
> **Service Discovery**: Confirmed endpoints for Redis, Elastic, and Kafka.
> **K8s Access**: The agent requires a ServiceAccount with appropriate RBAC permissions (ClusterRole mostly read-only) to query the K8s API.

## Proposed Changes

### Project Structure
- `app/`: Main application code.
    - `tools/`: K8s interaction tools.
    - `agent/`: LangGraph agent definition.
    - `server.py`: FastAPI entrypoint.
- `requirements.txt`: Dependencies.
- `Dockerfile`: Container image.
- `k8s/`: Kubernetes manifests for deployment.

### Architecture Overview
The system now consists of three main pillars:
1.  **The Sentinel Agent**: The brain, communicating with users and querying other components.
2.  **RAG Knowledge Base**: A daily ETL process that clones git repos (deployment configs, specialized docs), embeds them, and stores them for context retrieval.
3.  **Event Historian**: A streaming pipeline capturing transient K8s events into a persistent queryable history.

### K8s Tools (`app/tools/k8s_tools.py`)
Standard live-inspection tools:
- `get_cluster_health`, `list_pods`, `get_pod_details`, `get_live_events`, `get_logs`.

### Service Integrations
#### Redis Sentinel (`app/tools/redis_tools.py`)
- **Endpoint**: `redis-sentinel.redis-sentinel.svc.cluster.local:26379`
- **Tools**:
    - `GetRedisHealth`: Connects to Sentinel to verify Master is up and Slaves are replicating.
    - `GetRedisStats`: Checks connection counts/memory usage (critical for session management scaling).

#### Elasticsearch & APM (`app/tools/elastic_tools.py`)
- **Endpoint**: `elasticsearch-es-http.logging.svc.cluster.local:9200` / `apm-server-apm-http.logging.svc.cluster.local:8200`
- **Tools**:
    - `QueryAppLogs`: Queries ES indices for aggregated application logs (better for searching across time/pods than `kubectl logs`).
    - `GetAPMServiceHealth`: Queries APM data (if stored in ES) for high error rates or latency anomalies.

#### Kafka Inspection (`app/tools/kafka_tools.py`)
- **Requirement**: Install `kcat` (kafkacat) in the Docker image.
- **Tools**:
    - `GetKafkaMetadata`: Runs `kcat -L` to show brokers, topics, and partitions status.
    - `CheckConsumerLag`: detailed inspection of consumer group offsets (via `kcat` or `confluent-kafka`).
    - `PeekKafkaTopic`: Runs `kcat -C -c 5` to sample the last 5 messages from a topic (useful for debugging).

### RAG Knowledge Base (`app/rag/`)
We will use **Postgres with pgvector** (unified storage) or a simple functional equivalent.
- **Components**:
    - `VectorStore`: Interface for storing/retrieving embeddings.
    - `RepoIngester`: A Python script intended to run as a K8s `CronJob`.
        - Clones target repositories (configured via env vars).
        - Filters using `.dockerignore` style patterns (keep yaml, md, etc.).
        - Chunks text and computes embeddings (using a lightweight model like `all-MiniLM-L6-v2` to save the main LLM for generation, or the main LLM if capacity allows).
        - Upserts to VectorDB.
- **Agent Tool**: `lookup_deployment_policy(query)`: Searches the vector DB.

### Event Historian (`app/historian/`)
A streaming pipeline to ensure no event is lost.
- **Infrastructure**:
    - **Kafka**: `my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092`
    - **Postgres**: Permanent storage.
- **Components**:
    - `EventWatcher` (Deployment): Uses K8s `watch` API to stream events -> Kafka Topic `k8s-events`.
    - `EventArchiver` (Deployment): Consumes `k8s-events` -> Ingests into Postgres table `cluster_events`.
- **Agent Tool**: `query_event_history(filter, time_range)`: SQL query wrapper for natural language questions about the past.

### LangGraph Agent (`app/agent/graph.py`)
- **State**: Extended to include context retrieved from RAG.
- **Logic**:
    - If user asks about "history" or "past issues", route to `query_event_history`.
    - If user asks about "configuration" or "specs", route to `lookup_deployment_policy`.
    - If user asks about "now", use live K8s tools.

### Infrastructure Resource Design (`k8s/`)

#### 1. Sentinel Agent
- **Deployment**:
    - Replicas: 1 (Stateful interaction likely requires sticky sessions or single replica for simplicity initially).
    - Resources: `cpu: 500m`, `memory: 512Mi`.
    - Env Vars: `REDIS_URL`, `ELASTIC_URL`, `KAFKA_BOOTSTRAP`, `LLM_BASE_URL`.
- **ServiceAccount**:
    - `ClusterRole`: `view` (read-only) + specific rules for `events`, `nodes`, `pods/log`.
- **Service**: `ClusterIP` on port 80/8000.

#### 2. RAG System
- **CronJob** (`knowledge-refresher`):
    - Schedule: `0 3 * * *` (Daily at 3 AM).
    - Resources: `cpu: 1000m`, `memory: 1Gi` (Burst for embedding).
    - Volume: Ephemeral or PVC for git clones.
- **Vector Database**:
    - Use existing Postgres if available or deploy `pgvector` container.
    - ConfigMap: SQL init scripts for `vector` extension.

#### 3. Event Historian
- **Deployment** (`event-pipeline`):
    - **Container 1 (Producer)**: Python script using `kubernetes` watch -> Kafka.
    - **Container 2 (Consumer)**: Python script Kafka -> Postgres.
    - Resources: Low footprint (`cpu: 200m`, `memory: 256Mi` each).

#### 4. Verification Resources
- **Test Pod**: A temporary pod to run `curl` against the agent API for verification.

## Verification Plan

### Automated Tests
- We can write unit tests mocking the K8s API to ensure tools extract data correctly.

### Manual Verification
- Deploy to the cluster (or run locally with `~/.kube/config`).
- Ask questions:
    - "List pods in default namespace"
    - "Check status of node worker-1"
