**English** · [Русский](./README.ru.md)

# Food Talker — infrastructure

**The self-hosted retrieval stack behind [Food Talker](https://github.com/TroExol/food-talker),
a Telegram bot that finds restaurant dishes from natural-language queries.**

Everything runs locally through Docker Compose: vector storage, caching, a graph-based RAG engine and a
GPU reranker served by vLLM. No managed vector database, no hosted retrieval API.

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| **postgres** | `pgvector/pgvector:pg16` | Primary store plus vector similarity search over dish and menu embeddings |
| **redis** | `redis:7-alpine` | Caching and hot-path lookups |
| **qdrant** | `qdrant/qdrant` | Vector database used by the LightRAG pipeline |
| **lightrag** | `ghcr.io/hkuds/lightrag` | Graph-based RAG over the menu corpus |
| **vllm** | `vllm/vllm-openai` | Serves `BAAI/bge-reranker-v2-m3` behind an OpenAI-compatible API |
| **reranker** | local build | Reranking service that re-scores retrieval candidates |

Each service declares a healthcheck, and `postgres`/`redis` are always-on while the heavier GPU
services sit behind Compose profiles.

## Requirements

- Docker Engine + Docker Compose v2
- An NVIDIA GPU with the container toolkit installed — `reranker`, `lightrag` and `vllm` all request
  GPU resources
- Environment files in place: `food-talker/.env`, `food-talker-reranker/.env`, `LightRAG/.env`

## Running

```bash
# Core only — Postgres and Redis
docker compose up -d

# With the reranking service
docker compose --profile reranker up -d

# Full stack, including LightRAG, Qdrant and vLLM
docker compose --profile lightrag --profile reranker up -d
```

## Ports

| Service | Default port | Override |
|---------|--------------|----------|
| PostgreSQL | 5432 | `DB_PORT` |
| Redis | 6379 | `REDIS_PORT` |
| Reranker | 8000 | `SERVICE_PORT` |
| LightRAG | 9621 | `LIGHTRAG_PORT` |
| Qdrant | 6333 / 6334 | `QDRANT_PORT`, `QDRANT_GRPC_PORT` |
| vLLM | 8000 | `VLLM_PORT` |

> `reranker` and `vllm` both default to `8000` — set `VLLM_PORT` explicitly when running them together.

## Persistence

State is kept in named Docker volumes — `postgres_data`, `redis_data`, `qdrant_data`, `vllm_cache` —
so containers can be rebuilt without losing indexed data or downloaded model weights. LightRAG storage
and inputs are bind-mounted from `./LightRAG/data`.

## Related

- [food-talker](https://github.com/TroExol/food-talker) — the Telegram bot service itself
