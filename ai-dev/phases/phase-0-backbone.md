# Phase 0 — Backbone

**Goal:** a monorepo with all OSS services running locally and LiteLLM routing a test call to DeepSeek. **Time:** ~1 day. **Gate:** `manual-tests/mt-00-backbone.md` all green.

## Prerequisites (human does this, not the agent)
- Docker + docker-compose installed
- DeepSeek API key → put in root `.env` as `DEEPSEEK_API_KEY=...`
- `INTERNAL_SERVICE_TOKEN=<random 32+ chars>` in `.env`

## Tasks

### P0-T1 — Repo scaffold
Create exactly this structure (RULES.md R11):
```
crowdlens/
├── services/core-api/       # FastAPI skeleton: pyproject.toml, app/main.py with GET /health → {"status":"ok"}
├── services/adapters/       # empty package: pyproject.toml, adapters/__init__.py
├── forks/                   # empty for now (phase 1 fills)
├── frontend/                # Vite + React 18 + TS + Tailwind + shadcn/ui scaffold, one page: "CrowdLens" heading
├── infra/docker-compose.yml
├── infra/litellm/config.yaml
├── tests/integration/       # pytest skeleton
├── .env.example             # every key below, no values
└── AGENTS.md                # copy CONTEXT.md + RULES.md content into it verbatim
```
Deps allowed: `fastapi`, `uvicorn`, `httpx`, `sqlalchemy`, `alembic`, `psycopg[binary]`, `pydantic`, `pytest`. Frontend: `react`, `vite`, `typescript`, `tailwindcss`, `shadcn-ui` per its official init.
**Done when:** `cd services/core-api && uvicorn app.main:app` serves `/health`; `cd frontend && npm run dev` renders the page; both paste output.

### P0-T2 — docker-compose
`infra/docker-compose.yml` with services: `postgres:16` (db `crowdlens`), `neo4j:5-community` (ports 7474/7687), `redis:7`, `minio` (+ `minio-init` bucket creation job for bucket `crowdlens-artifacts`), `litellm` (config mounted from `infra/litellm/config.yaml`, port 4000), `langfuse` + its postgres + clickhouse per official Langfuse self-host compose. Named volumes for all state. All on network `crowdlens-net`.
**Done when:** `docker compose -f infra/docker-compose.yml up -d` → all containers healthy; `curl localhost:4000/health` OK; MinIO console on 9001 shows the bucket.

### P0-T3 — LiteLLM model aliases
`infra/litellm/config.yaml` defines these aliases (`deepseek/` provider for the first two):
- `swarm-model` → the DeepSeek chat/flash model, `cache: true`
- `report-model` → the DeepSeek pro/reasoner-class model
- `embed-model` → an embedding model (DeepSeek has no embeddings endpoint — point this alias at any LiteLLM-supported embedding provider the human configures; not needed until phase 5, define the alias now so phases don't drift)
Plus: `general_settings: master_key: ${LITELLM_MASTER_KEY}`. Human adds `LITELLM_MASTER_KEY` to `.env`.
**Done when:** `curl localhost:4000/v1/chat/completions -H "Authorization: Bearer $LITELLM_MASTER_KEY" -d '{"model":"swarm-model","messages":[{"role":"user","content":"say ok"}]}'` returns a completion, and the Langfuse/LiteLLM logs show the call.

### P0-T4 — Database migrations
Alembic init in `services/core-api`; first migration creates every table in `contracts/data-model.md` verbatim (including CHECK constraints and indexes). SQLAlchemy models in `app/models.py` matching 1:1.
**Done when:** `alembic upgrade head` runs clean against the compose postgres; `\dt` in psql lists all 12 tables; `alembic downgrade base` also clean.

### P0-T5 — Core API health + config wiring
`core-api` reads `.env` via pydantic-settings: `DATABASE_URL`, `LITELLM_BASE_URL=http://localhost:4000`, `LITELLM_MASTER_KEY`, `INTERNAL_SERVICE_TOKEN`, `AUTH_DISABLED=true`. Add `GET /api/v1/health` returning `{"api":"ok","db":"ok","litellm":"ok"}` — actually probe DB (`SELECT 1`) and LiteLLM (`/health`).
**Done when:** endpoint returns all-ok with compose stack up; returns `503` with the failing component named when postgres is stopped (prove it by stopping the container).

## Stop conditions
- Langfuse's self-host compose layout changed upstream → stop, paste the upstream README link, ask.
- Any port conflict → stop, report, wait for a port decision. Do not silently remap.

## Manual test
Open `../manual-tests/mt-00-backbone.md`, execute every step, record results. Phase 1 starts only when it's all green.
