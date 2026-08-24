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

**Files:**
- `services/core-api/pyproject.toml` — project `crowdlens-core-api`; only the deps above, pinned (R8). `[tool.pytest.ini_options]` not needed yet.
- `services/core-api/app/__init__.py` — empty.
- `services/core-api/app/main.py` — `app = FastAPI(title="crowdlens-core-api")` and `@app.get("/health") def health() -> dict: return {"status": "ok"}`. Nothing else.
- `services/core-api/tests/test_health.py` — see below.
- `services/adapters/pyproject.toml` + `services/adapters/adapters/__init__.py` — empty package, no deps of its own yet.
- `frontend/` — scaffold via the official Vite react-ts template, then Tailwind per tailwindcss.com docs for the installed major version, then shadcn/ui per ui.shadcn.com init docs (order matters: Tailwind first). One page in `frontend/src/App.tsx` rendering an `<h1>CrowdLens</h1>`.
- `infra/docker-compose.yml` — empty placeholder (`services:` only); P0-T2 fills it.
- `infra/litellm/config.yaml` — empty placeholder; P0-T3 fills it.
- `tests/integration/__init__.py` + `tests/integration/conftest.py` — empty skeleton.
- `.env.example` — keys only, no values: `DEEPSEEK_API_KEY`, `LITELLM_MASTER_KEY`, `INTERNAL_SERVICE_TOKEN`, `DATABASE_URL`, `LITELLM_BASE_URL`, `AUTH_DISABLED`, plus service creds used by compose (`POSTGRES_PASSWORD`, `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, `NEO4J_AUTH`).
- `AGENTS.md` — verbatim copy of `ai-dev/CONTEXT.md` + `ai-dev/RULES.md` content.

**Tests:** `services/core-api/tests/test_health.py`
- `test_health_returns_ok` — fastapi `TestClient` GET `/health` → 200, body `{"status":"ok"}`.

**Edge cases:**
- `uvicorn app.main:app` only resolves if run from `services/core-api/` — the done-criteria `cd` is load-bearing; document it in the file header comment if the agent adds a README later.
- `.env.example` must exist and be non-empty even though `.env` is gitignored; mt-00 prereq depends on the human knowing the keys.
- shadcn/ui init fails on an unconfigured Tailwind setup; run its CLI only after Tailwind builds clean.
- `crowdlens/` in the tree is the logical project root; do not create a literal nested `crowdlens/crowdlens/` directory.

**Failure modes (cheap LLM):**
- Inventing extra endpoints, routers, or directories "for later" — the tree above is exhaustive (R11); anything more is scope creep.
- Adding unlisted deps (pydantic-settings, sqlalchemy-utils, etc.) "because they're standard" — R8; `pydantic-settings` is first needed in P0-T5, add it then.
- Scaffolding shadcn/ui from memory with a stale CLI invocation — run the official init for the current version instead of guessing flags.
- Writing `AGENTS.md` as a paraphrase — it must be a verbatim copy of the two source files.

### P0-T2 — docker-compose
`infra/docker-compose.yml` with services: `postgres:16` (db `crowdlens`), `neo4j:5-community` (ports 7474/7687), `redis:7`, `minio` (+ `minio-init` bucket creation job for bucket `crowdlens-artifacts`), `litellm` (config mounted from `infra/litellm/config.yaml`, port 4000), `langfuse` + its postgres + clickhouse per official Langfuse self-host compose. Named volumes for all state. All on network `crowdlens-net`.
**Done when:** `docker compose -f infra/docker-compose.yml up -d` → all containers healthy; `curl localhost:4000/health` OK; MinIO console on 9001 shows the bucket.

**Files:**
- `infra/docker-compose.yml` — the only file. Service checklist:
  - `postgres`: image `postgres:16`, env `POSTGRES_DB=crowdlens` / `POSTGRES_USER` / `POSTGRES_PASSWORD`, healthcheck `pg_isready -U $POSTGRES_USER -d crowdlens`, port 5432.
  - `neo4j`: image `neo4j:5-community`, env `NEO4J_AUTH=neo4j/<password>`, ports 7474/7687, healthcheck on the bolt or HTTP port.
  - `redis`: image `redis:7`, port 6379, healthcheck `redis-cli ping`.
  - `minio`: image `minio/minio` (pin a RELEASE tag, not `latest`), ports 9000/9001, command `server /data --console-address ":9001"`.
  - `minio-init`: image `minio/mc`, one-shot job (`restart: "no"`) that runs `mc alias set` + `mc mb` for bucket `crowdlens-artifacts`; `depends_on: minio: condition: service_healthy`.
  - `litellm`: image `ghcr.io/berriai/litellm` (pinned tag), `--config /app/config.yaml` mounted from `infra/litellm/config.yaml`, port 4000, env `DEEPSEEK_API_KEY` + `LITELLM_MASTER_KEY` passed through.
  - `langfuse` stack: copy the service set from the official Langfuse self-hosting docker-compose (langfuse.com/self-hosting) — do not reconstruct it from memory; upstream layout drift is a named stop condition.
  - Top-level `volumes:` (one per stateful service) and `networks: crowdlens-net`.
- `.env` (human-owned) — gains `POSTGRES_PASSWORD`, `MINIO_ROOT_USER/PASSWORD`, `NEO4J_AUTH`, Langfuse secrets per upstream compose.

**Tests:** none automated here — mt-00 steps 1, 2, 5 are the tests.

**Edge cases:**
- Re-running `up` after a broken first attempt: stale named volumes keep half-initialized state (e.g. postgres with a different password) — document `docker compose down -v` as the reset, never delete volumes silently (R9).
- `minio-init` is expected to exit 0 and stay `Exited`; it must not block `ps` health of the rest and must be idempotent (`mc mb` on an existing bucket exits non-zero — guard or accept).
- Neo4j first boot takes 30–60s; healthchecks need `start_period` or mt-00 step 1 flakes.
- Langfuse upstream compose may add services (e.g. a separate worker); copy the whole set, don't cherry-pick.

**Failure modes (cheap LLM):**
- Inventing Langfuse env var names from memory — its required secrets change between versions; copy them from the upstream compose file, else stop-condition #1 applies.
- Using `latest` tags — R8 requires pins; a `latest` pull mid-project breaks reproducibility.
- Bind-mounting host paths instead of named volumes (permission churn, platform-specific paths).
- Forgetting `crowdlens-net` on one service so `bettafish`/`core-api` can't resolve `litellm` later — every service goes on the network.
- Silently remapping a conflicting host port — that is stop-condition #2; report instead.

### P0-T3 — LiteLLM model aliases
`infra/litellm/config.yaml` defines these aliases (`deepseek/` provider for the first two):
- `swarm-model` → the DeepSeek chat/flash model, `cache: true`
- `report-model` → the DeepSeek pro/reasoner-class model
- `embed-model` → an embedding model (DeepSeek has no embeddings endpoint — point this alias at any LiteLLM-supported embedding provider the human configures; not needed until phase 5, define the alias now so phases don't drift)
Plus: `general_settings: master_key: ${LITELLM_MASTER_KEY}`. Human adds `LITELLM_MASTER_KEY` to `.env`.
**Done when:** `curl localhost:4000/v1/chat/completions -H "Authorization: Bearer $LITELLM_MASTER_KEY" -d '{"model":"swarm-model","messages":[{"role":"user","content":"say ok"}]}'` returns a completion, and the Langfuse/LiteLLM logs show the call.

**Files:**
- `infra/litellm/config.yaml` — the only file. Shape (verify field names against the LiteLLM proxy config docs at docs.litellm.ai before writing — config schema drifts between versions):
  - `model_list:` — three entries, each with `model_name` = the alias and `litellm_params:` with `model:` = provider-prefixed upstream model and `api_key: os.environ/...`.
  - `swarm-model`: `model: deepseek/<chat-model>` — the exact upstream model string comes from the DeepSeek platform docs (api-docs.deepseek.com), not from memory.
  - `report-model`: `model: deepseek/<reasoner-class-model>` — same source.
  - `embed-model`: placeholder entry for a human-chosen embedding provider; the alias line exists now, the working `api_key` env is documented in `.env.example` as optional until phase 5.
  - Response caching for `swarm-model`: enable via the proxy's cache settings (`litellm_settings`) per the LiteLLM caching docs — confirm the current config key name there.
  - `general_settings: master_key: ${LITELLM_MASTER_KEY}` (env substitution as shown).

**Tests:** none automated — mt-00 steps 3, 4, 12 are the tests (both aliases route; both calls logged with token counts).

**Edge cases:**
- LiteLLM supports both `${VAR}` and `os.environ/VAR` substitution depending on config section — mixing them silently yields literal strings as API keys; test with the mt-00 curl before declaring done.
- `/health` must answer without the master key; `/v1/*` must reject without it — spot-check a no-auth call returns 401 so keys aren't decorative.
- `embed-model` with a missing/unset provider key must not prevent the proxy from booting; if it does, comment the params and keep the alias documented — then report the drift.

**Failure modes (cheap LLM):**
- Hallucinating upstream model IDs (e.g. inventing `deepseek-v4-flash` as a provider string) — the marketing name and the API model string differ; copy from DeepSeek's official docs.
- Omitting the `deepseek/` provider prefix so LiteLLM misroutes.
- Putting `master_key` under `model_list` or `litellm_settings` instead of `general_settings`.
- "Testing" by trusting container logs instead of running the exact mt-00 curl (R7).

### P0-T4 — Database migrations
Alembic init in `services/core-api`; first migration creates every table in `contracts/data-model.md` verbatim (including CHECK constraints and indexes). SQLAlchemy models in `app/models.py` matching 1:1.
**Done when:** `alembic upgrade head` runs clean against the compose postgres; `\dt` in psql lists all 12 tables; `alembic downgrade base` also clean.

**Files:**
- `services/core-api/alembic.ini` — stock; `sqlalchemy.url` left unset (env.py injects it).
- `services/core-api/alembic/env.py` — reads `DATABASE_URL` from the environment (settings module arrives in P0-T5; read `os.environ` directly for now).
- `services/core-api/alembic/versions/0001_initial_schema.py` — `upgrade()` runs `CREATE EXTENSION IF NOT EXISTS citext;` first, then creates the 13 phase-0–2 tables from `contracts/data-model.md` **by copying the DDL verbatim** (workspaces, users, workspace_members, projects, seed_documents, grounding_jobs, collected_items, baseline_reports, persona_panels, simulations, sim_runs, verdicts, cost_ledger) plus indexes `idx_items_job`, `idx_items_source`, `idx_ledger_project`. `downgrade()` drops in reverse FK order and drops the extension. Later-phase tables (audit_events, reports, …) are **not** created here.
- `services/core-api/app/models.py` — SQLAlchemy 2.0-style declarative models, 1:1 with the DDL: same column names, same nullability, same CHECK/UNIQUE constraints via `CheckConstraint`/`UniqueConstraint`, `uuid` PKs with `server_default=gen_random_uuid()`.

**Tests:** `tests/integration/test_migrations.py` (runs against the compose postgres; mark with a fixture that skips if `DATABASE_URL` unreachable):
- `test_upgrade_creates_all_tables` — upgrade head → information_schema shows exactly the 13 tables.
- `test_check_constraints_enforced` — inserting `projects.status='bogus'` raises `IntegrityError`.
- `test_unique_constraints_enforced` — duplicate `(project_id, version)` in `seed_documents` raises.
- `test_downgrade_base_clean` — downgrade → zero app tables remain; upgrade again → green (mirrors mt-00 step 7).
- `test_indexes_exist` — the three named indexes present in `pg_indexes`.

**Edge cases:**
- `citext` must exist before `users` is created — ordering inside one migration matters.
- `gen_random_uuid()` is core in PG16; do **not** add `pgcrypto` (data-model.md says so explicitly).
- Downgrade order: `cost_ledger`, `verdicts`, `sim_runs`, … before `workspaces` — FK violations on downgrade are the classic half-applied state mt-00 step 7 hunts for.
- `numeric(10,2)` vs `numeric(10,4)` vs `numeric(10,6)` precisions differ per column — copy, don't "simplify".
- `grounding_jobs.id`, `baseline_reports.id`, `simulations.id`, `sim_runs.id` have **no** default — core-api/adapter supplies them; models.py must not add `gen_random_uuid()` there.

**Failure modes (cheap LLM):**
- Regenerating the schema from the table list in prose instead of copying the SQL block — misses CHECK constraints, `UNIQUE (project_id, version)`, index names. Verbatim copy only.
- Creating the later-phase tables "while we're here" — they belong to their phases' migrations (data-model.md is explicit); doing so desyncs phase 3–8 tasks.
- Letting Alembic autogenerate from models.py and shipping that — autogen misses `citext`, server defaults, and CHECKs; write 0001 by hand from the contract.
- models.py drifting from DDL (nullable mismatch, wrong types) — the 1:1 requirement exists so phase-1+ code can trust the models.

### P0-T5 — Core API health + config wiring
`core-api` reads `.env` via pydantic-settings: `DATABASE_URL`, `LITELLM_BASE_URL=http://localhost:4000`, `LITELLM_MASTER_KEY`, `INTERNAL_SERVICE_TOKEN`, `AUTH_DISABLED=true`. Add `GET /api/v1/health` returning `{"api":"ok","db":"ok","litellm":"ok"}` — actually probe DB (`SELECT 1`) and LiteLLM (`/health`).
**Done when:** endpoint returns all-ok with compose stack up; returns `503` with the failing component named when postgres is stopped (prove it by stopping the container).

**Files:**
- `services/core-api/app/config.py` — `class Settings(BaseSettings)` (pydantic-settings; add the dep now) with exactly the five fields above; `AUTH_DISABLED: bool = True`. `@lru_cache def get_settings() -> Settings`.
- `services/core-api/app/db.py` — SQLAlchemy engine + `SessionLocal`; `def probe_db() -> bool` running `SELECT 1` with a short connect timeout.
- `services/core-api/app/litellm.py` — `async def probe_litellm() -> bool`: `GET {LITELLM_BASE_URL}/health` via httpx with `Authorization: Bearer {LITELLM_MASTER_KEY}`, timeout ≤3s.
- `services/core-api/app/main.py` — keep `GET /health`; add `GET /api/v1/health` that runs both probes and returns 200 with `{"api":"ok","db":"ok","litellm":"ok"}` or **503** with the failing component(s) named, e.g. `{"api":"ok","db":"down","litellm":"ok"}`.
- `services/core-api/pyproject.toml` — add pinned `pydantic-settings`.

**Tests:** `services/core-api/tests/test_health.py` (extend the P0-T1 file)
- `test_health_returns_ok` — unchanged from P0-T1.
- `test_api_health_all_ok` — probes mocked to succeed → 200, all-ok body.
- `test_api_health_db_down` — `probe_db` mocked False → 503, body has `db` != "ok", others "ok".
- `test_api_health_litellm_down` — `probe_litellm` mocked False → 503, `litellm` named.
- `test_settings_require_database_url` — Settings without `DATABASE_URL` raises validation error.

**Edge cases:**
- Probes need timeouts — a stopped postgres can hang a connection attempt far longer than mt-00 step 9's 5s wait; set connect timeout ≤2s so 503 returns promptly.
- `probe_litellm` treats any non-2xx (including 401 from a bad master key) as "down" — wrong key must surface as `litellm: down`, not as a crash.
- Exception paths inside probes return False, they never propagate (a probe must not 500 the health endpoint) — but log the original exception verbatim (R10).

**Failure modes (cheap LLM):**
- Returning `"ok"` unconditionally and letting mt-00 step 9 catch it — the probes must be real; fake success is the exact R10 violation this task exists to prevent.
- Caching probe results — health must reflect now, not the last check.
- Reading `.env` ad hoc with `os.getenv` sprinkled across modules — all config goes through `Settings` so phase 3's auth config has one home.
- Naming the 503 body differently per failure — the mt-00 check is "body names `db`"; keep component keys stable (`api`/`db`/`litellm`).

## Stop conditions
- Langfuse's self-host compose layout changed upstream → stop, paste the upstream README link, ask.
- Any port conflict → stop, report, wait for a port decision. Do not silently remap.

## Manual test
Open `../manual-tests/mt-00-backbone.md`, execute every step, record results. Phase 1 starts only when it's all green.
