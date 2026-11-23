# K8s Sentinel Agent

Agente de IA para monitoreo conversacional de clústeres Kubernetes. Responde preguntas en lenguaje natural sobre el estado del cluster usando un modelo LLM local (Qwen3-4b) y herramientas que consultan la API de Kubernetes, Redis, Elasticsearch y Kafka.

## Stack

- **Agent framework:** LangGraph
- **LLM:** Qwen3-4b (local, via vLLM o qwen-agent)
- **API:** FastAPI (compatible con OpenWebUI)
- **Kubernetes:** client Python + kubectl
- **Infra:** Redis, Elasticsearch, Kafka/Redpanda, PostgreSQL (pgvector)

## Arquitectura

Tres pilares:

1. **Sentinel Agent** — El cerebro. Interactúa con el usuario y consulta los otros componentes.
2. **RAG Knowledge Base** — Pipeline ETL diario que clona repos de configuración (deployment configs, docs), los chunka, genera embeddings y los almacena para retrieval contextual.
3. **Event Historian** — Pipeline de streaming que captura eventos efímeros de Kubernetes (watch) y los persiste en una base de datos consultable.

## Herramientas disponibles

| Herramienta | Descripción |
|-------------|-------------|
| `get_cluster_health` | Estado general del cluster (nodos, pods, alertas) |
| `list_pods` | Listar pods con filtros (namespace, label, estado) |
| `get_pod_details` | Detalle de un pod específico |
| `get_live_events` | Eventos en tiempo real del cluster |
| `get_logs` | Logs de un pod o namespace |
| `query_knowledge_base` | Búsqueda RAG sobre docs y configs |
| `query_event_history` | Búsqueda sobre eventos históricos |
| `redis_tools` | Estado de Redis Sentinel, métricas de master/slaves |
| `elasticsearch_tools` | Query de logs, APM, error rates |
| `kafka_tools` | Metadata de topics, consumer lag, sample messages |

## Estructura del proyecto

```
├── app/
│   ├── agent/          # Definición del agente LangGraph
│   ├── tools/          # Herramientas de interacción (K8s, Redis, ES, Kafka)
│   └── server.py       # Entrypoint FastAPI
├── k8s/                # Manifiestos Kubernetes (Deployment, Service, RBAC)
├── requirements.txt
├── Dockerfile
└── docs/
    ├── implementation_plan.md
    ├── infra_spec.md
    └── task.md
```

## Requisitos

- Kubernetes cluster con RBAC (ServiceAccount con ClusterRole read-only)
- Redis (Sentinel)
- Elasticsearch
- Kafka o Redpanda
- PostgreSQL con pgvector
- Modelo LLM local servido vía vLLM

## RBAC

El agente requiere un ServiceAccount con permisos de solo lectura:

- `core`: pods, nodes, namespaces, services, events, configmaps (get, list, watch)
- `core`: pods/log (get)
- `apps`: deployments, statefulsets, daemonsets (get, list, watch)
- `batch`: jobs, cronjobs (get, list, watch)
- `metrics.k8s.io`: pods, nodes (get, list)

## Estado del proyecto

En fase de diseño y planificación. Ver [docs/implementation_plan.md](docs/implementation_plan.md) y [docs/infra_spec.md](docs/infra_spec.md) para el detalle.

## Autor

Juan Pablo Garriz
