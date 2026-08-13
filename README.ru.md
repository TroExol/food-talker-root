[English](./README.md) · **Русский**

# Food Talker — инфраструктура

**Self-hosted retrieval-стек, на котором работает [Food Talker](https://github.com/TroExol/food-talker) — Telegram-бот, находящий блюда в ресторанах по запросам на естественном языке.**

Всё поднимается локально через Docker Compose: векторное хранилище, кеш, граф-RAG движок и GPU-реранкер на vLLM. Без managed векторной БД и без внешнего retrieval-API.

## Сервисы

| Сервис | Образ | Назначение |
|--------|-------|-----------|
| **postgres** | `pgvector/pgvector:pg16` | Основное хранилище плюс векторный поиск по эмбеддингам блюд и меню |
| **redis** | `redis:7-alpine` | Кеширование и горячие запросы |
| **qdrant** | `qdrant/qdrant` | Векторная БД для пайплайна LightRAG |
| **lightrag** | `ghcr.io/hkuds/lightrag` | Граф-RAG по корпусу меню |
| **vllm** | `vllm/vllm-openai` | Отдаёт `BAAI/bge-reranker-v2-m3` через OpenAI-совместимый API |
| **reranker** | локальная сборка | Сервис реранкинга, пересортировывающий кандидатов из поиска |

У каждого сервиса свой healthcheck. `postgres` и `redis` работают всегда, тяжёлые GPU-сервисы спрятаны за Compose-профилями.

## Требования

- Docker Engine + Docker Compose v2
- NVIDIA GPU с установленным container toolkit — `reranker`, `lightrag` и `vllm` запрашивают GPU
- Файлы окружения на месте: `food-talker/.env`, `food-talker-reranker/.env`, `LightRAG/.env`

## Запуск

```bash
# Только ядро — Postgres и Redis
docker compose up -d

# С сервисом реранкинга
docker compose --profile reranker up -d

# Полный стек, включая LightRAG, Qdrant и vLLM
docker compose --profile lightrag --profile reranker up -d
```

## Порты

| Сервис | Порт по умолчанию | Переменная |
|--------|-------------------|-----------|
| PostgreSQL | 5432 | `DB_PORT` |
| Redis | 6379 | `REDIS_PORT` |
| Reranker | 8000 | `SERVICE_PORT` |
| LightRAG | 9621 | `LIGHTRAG_PORT` |
| Qdrant | 6333 / 6334 | `QDRANT_PORT`, `QDRANT_GRPC_PORT` |
| vLLM | 8000 | `VLLM_PORT` |

> `reranker` и `vllm` по умолчанию оба на `8000` — при одновременном запуске задавать `VLLM_PORT` явно.

## Хранение данных

Состояние живёт в именованных Docker volumes — `postgres_data`, `redis_data`, `qdrant_data`, `vllm_cache` — поэтому контейнеры можно пересобирать без потери проиндексированных данных и скачанных весов модели. Хранилище и входные данные LightRAG монтируются из `./LightRAG/data`.

## Связанное

- [food-talker](https://github.com/TroExol/food-talker) — сам сервис Telegram-бота
