# Lesson 13 - Контейнери для AI

Здача: `alexxmore`

## Що додано

- `Dockerfile.naive` - базовий образ на `python:3.11` з `COPY . .` та `pip install`.
- `Dockerfile` - multi-stage образ на `python:3.11-slim`, non-root користувач у runtime та `HEALTHCHECK` для `/health`.
- `.dockerignore` - прибирає з build context локальний `.env`, `.venv`, кеші, тести та службові файли.
- `docker-compose.yml` - застосунок + Langfuse + Qdrant + Redis + Postgres для зберігання даних Langfuse.

## Локальний запуск

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r app\requirements.txt pytest pytest-asyncio
Copy-Item .env.example .env
```

У файлі `.env` потрібно вказати `OPENAI_API_KEY`.

У моєму Docker Desktop середовищі запити до OpenAI з Linux-контейнерів потребували:

```env
OPENAI_VERIFY_SSL=false
```

У `.env.example` значення за замовчуванням лишається безпечним:

```env
OPENAI_VERIFY_SSL=true
```

## Docker-команди

```powershell
docker build -f Dockerfile.naive -t alexxmore-rag:naive .
docker build -f Dockerfile -t alexxmore-rag:multi-stage .
docker compose up -d --build
```

Запит до `/ask`:

```powershell
Invoke-RestMethod -Method Post http://127.0.0.1:8000/ask `
  -ContentType 'application/json' `
  -Body '{"question":"What is a vector database?"}'
```

## Метрики

Виміряно на Docker Desktop 29.4.3. Час збірки вимірювався через `docker build --no-cache`, базові образи вже були доступні локально.

| Метрика | Naive | Multi-stage |
|---|---:|---:|
| Розмір образу | 1.76 GB | 367 MB |
| Час збірки | 19.19s | 24.48s |
| Перезбірка після зміни коду | 18.15s | 2.45s |
| Cold start до `/health=ok` | 3.13s | 2.60s |

## Докази

`docker images`:

```text
REPOSITORY      TAG           SIZE
alexxmore-rag   multi-stage   367MB
alexxmore-rag   naive         1.76GB
```

`docker compose ps`:

```text
NAME                   IMAGE                       SERVICE    STATUS
alexxmore-app-1        alexxmore-rag:multi-stage   app        Up (healthy)
alexxmore-langfuse-1   langfuse/langfuse:2         langfuse   Up
alexxmore-postgres-1   postgres:16-alpine          postgres   Up (healthy)
alexxmore-qdrant-1     qdrant/qdrant:v1.12.6       qdrant     Up
alexxmore-redis-1      redis:7-alpine              redis      Up
```

`POST /ask`:

```json
{
  "answer": "A vector database stores embeddings and performs approximate nearest-neighbor search to retrieve semantically similar documents. Examples include Qdrant, pgvector, Chroma, and Pinecone.",
  "sources": [
    {
      "question": "What is a vector database?",
      "answer": "A vector database stores embeddings and performs approximate nearest-neighbor search to retrieve semantically similar documents. Examples: Qdrant, pgvector, Chroma, Pinecone."
    }
  ]
}
```
