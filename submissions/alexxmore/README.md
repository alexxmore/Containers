# Lesson 13 - Containers for AI

Submission: `alexxmore`

## What is included

- `Dockerfile.naive` - baseline image from `python:3.11` with `COPY . .` and `pip install`.
- `Dockerfile` - multi-stage image from `python:3.11-slim`, non-root runtime user, and `/health` healthcheck.
- `.dockerignore` - excludes local env, venv, caches, tests, and metadata from the build context.
- `docker-compose.yml` - app + Langfuse + Qdrant + Redis + Postgres for Langfuse storage.

## Local setup

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r app\requirements.txt pytest pytest-asyncio
Copy-Item .env.example .env
```

Set `OPENAI_API_KEY` in `.env`.

In this Docker Desktop environment, OpenAI calls from Linux containers required:

```env
OPENAI_VERIFY_SSL=false
```

The default in `.env.example` remains `OPENAI_VERIFY_SSL=true`.

## Docker commands

```powershell
docker build -f Dockerfile.naive -t alexxmore-rag:naive .
docker build -f Dockerfile -t alexxmore-rag:multi-stage .
docker compose up -d --build
```

Ask endpoint:

```powershell
Invoke-RestMethod -Method Post http://127.0.0.1:8000/ask `
  -ContentType 'application/json' `
  -Body '{"question":"What is a vector database?"}'
```

## Metrics

Measured on Docker Desktop 29.4.3. Build time uses `docker build --no-cache` with base images already available locally.

| Metric | Naive | Multi-stage |
|---|---:|---:|
| Image size | 1.76 GB | 367 MB |
| Build time | 19.19s | 24.48s |
| Rebuild after code change | 18.15s | 2.45s |
| Cold start to `/health=ok` | 3.13s | 2.60s |

## Evidence

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
