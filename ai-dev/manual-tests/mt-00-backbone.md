# MT-00 — Backbone

**Prereq:** `.env` filled (DeepSeek key, LiteLLM master key, internal token). Run from repo root.

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | `docker compose -f infra/docker-compose.yml up -d` | All containers start; `docker compose ps` shows healthy/running for postgres, neo4j, redis, minio, litellm, langfuse | |
| 2 | `curl localhost:4000/health` | LiteLLM healthy | |
| 3 | `curl localhost:4000/v1/chat/completions -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H "Content-Type: application/json" -d '{"model":"swarm-model","messages":[{"role":"user","content":"Reply with exactly: ok"}]}'` | JSON completion containing "ok" | |
| 4 | Same curl with `"model":"report-model"` | Completion succeeds (different model alias routes) | |
| 5 | Open `http://localhost:9001` (MinIO console) | Bucket `crowdlens-artifacts` exists | |
| 6 | `cd services/core-api && alembic upgrade head` | Runs clean; then `docker exec <postgres> psql -U postgres -d crowdlens -c '\dt'` lists 13 tables: workspaces, users, workspace_members, projects, seed_documents, grounding_jobs, collected_items, baseline_reports, persona_panels, simulations, sim_runs, verdicts, cost_ledger (+ alembic_version) | |
| 7 | `alembic downgrade base && alembic upgrade head` | Both directions clean — no half-applied state | |
| 8 | `curl localhost:8000/api/v1/health` | `{"api":"ok","db":"ok","litellm":"ok"}` | |
| 9 | `docker compose stop postgres`, wait 5s, re-run step 8 | `503`, body names `db` as the failure | |
| 10 | `docker compose start postgres` | Health returns to all-ok | |
| 11 | `cd frontend && npm run dev`, open the page | "CrowdLens" heading renders, no console errors | |
| 12 | Check LiteLLM logs/spend UI for the step 3–4 calls | Both calls logged with token counts — cost tracking works from day one | |

**Sign-off:** all PASS → proceed to phase 1. Any FAIL → failure recovery session before continuing.
