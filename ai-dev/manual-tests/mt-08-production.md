# MT-08 — Production Hardening (launch gate)

**Prereq:** MT-07 green. A fresh VM (or clean VPS) available for the deploy test. This checklist gates launch.

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | **Temporal durability:** start an ensemble on staging; mid-run, `docker compose kill -s 9 worker`; restart | Ensemble resumes from checkpoint and completes; verdict identical in shape to a non-interrupted run | |
| 2 | **Fresh deploy:** on the clean VM, follow `infra/DEPLOY.md` exactly | One command → full stack healthy behind TLS on the domain; no manual fixes needed (any fix = doc bug = FAIL until documented) | |
| 3 | **Secrets audit:** `grep -r` the repo + VM for real-looking keys; confirm `.env` files absent from the prod host (env injection only) | Nothing found; CORS allows only the prod domain | |
| 4 | Rotate `INTERNAL_SERVICE_TOKEN` per DEPLOY.md | Stack healthy after rotation (proves rotation is possible) | |
| 5 | **Alerting:** `docker stop bettafish` → within 5 min an alert fires (Grafana/Alertmanager → your channel); restart it | Alert fired and cleared; Langfuse dashboard shows the eval scores | |
| 6 | **Restore drill:** take last night's backup; restore into a scratch stack per `infra/restore.sh`; hit `/api/v1/health` and spot-check one project | Restore works end-to-end — an untested backup is not a backup | |
| 7 | **Security pass:** `pip-audit` + `npm audit` output clean/documented; route-table RBAC test green; rate limit proof (burst 150 requests → 429s); `/legal/licenses` lists both AGPL forks with source links | All green | |
| 8 | **SSO:** enable SAML on a test workspace with a test IdP; login as an IdP user | Lands in the right workspace with the mapped role; audit log records it | |
| 9 | **On-prem profile:** bring the stack up with the `profiles/onprem` overlay pointing LiteLLM at a local Ollama; run one small decision cycle | Cycle completes with zero external LLM calls (verify in network logs); license file required at startup; removing it → clean refusal | |
| 10 | **Load test:** `locust -f tests/load/locustfile.py` — 50 WS viewers + 10 concurrent decision cycles | p95 API < 500ms; WS latency < 2s; zero 5xx | |
| 11 | **Full regression:** re-run MT-00 → MT-07 checklists against the staging environment | Every step still PASS | |

**Sign-off:** all PASS = production launch per PBR roadmap M1–M5 complete. Keep this file's results — enterprise buyers will ask for the DR drill (step 6) and security pass (step 7) evidence.
