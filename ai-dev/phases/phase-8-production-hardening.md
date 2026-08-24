# Phase 8 — Production Hardening & Enterprise

**Goal:** from "works on my machine" to deployable product: Temporal orchestration, prod infra, observability, security, SSO, on-prem profile. **Gate:** `manual-tests/mt-08-production.md` green — this is the launch checklist.

## Prerequisites
- Phase 7 green. Human decisions needed upfront: production domain, cloud host, SMTP/WhatsApp providers for Novu, Lago plans created.

## Tasks

### P8-T1 — Temporal migration
Move `ensemble` orchestration, grounding polling, monitoring schedules, and export jobs onto Temporal per `contracts/platform-services.md` §Temporal. Same business logic, durable execution. The adaptive-ensemble integration tests from P2-T5 must pass unchanged against the Temporal implementation (behavior-preserving refactor only — R6).
**Done when:** kill -9 the worker mid-ensemble → `docker compose start worker` → ensemble resumes from its last checkpoint and completes; P2-T5 tests green.

**Files:**
- `app/temporal/__init__.py`, `app/temporal/workflows.py`, `app/temporal/activities.py`, `app/temporal/worker.py` — new; task queue `crowdlens`.
- `services/core-api/` compose entry gains a `worker` service (same image, different command) + `temporal` + `temporal-ui` services.
- Old code paths (`app/services/ensemble.py` background task, APScheduler in `app/services/scheduler.py`, phase-5 export background tasks) become thin entry points that start workflows. Delete the APScheduler dependency only after `MonitoringWorkflow` passes mt-07 steps 1–2 against Temporal.

**Workflows (names fixed by platform-services §Temporal):**
```python
@workflow.defn
class EnsembleWorkflow:
    @workflow.run
    async def run(self, simulation_id: str) -> EnsembleResult: ...
    @workflow.signal
    def halt_budget(self) -> None: ...

@workflow.defn
class GroundingWorkflow:    # input {job_id}, signal cancel, activity heartbeat 30s
@workflow.defn
class MonitoringWorkflow:   # input {project_id, cron}, signal run_now
@workflow.defn
class ExportWorkflow:       # input {report_id, format}
```
Activities (thin wrappers over the existing services — logic moves, behavior doesn't): `launch_ensemble_run`, `poll_simulation_status`, `check_budget`, `upsert_items`, `render_export`. Retry policy on all: 5 attempts, exponential 10s→5min. Idempotency keys = entity UUIDs (the `simulation_id`/`job_id`/`report_id` already are UUIDs — use them as Temporal workflow ids). Ensemble child workflow per run; checkpoint after every round-poll (contract wording) — in practice: persist each round-poll result as an activity result so replay reconstructs state.

**Temporal determinism rules (the classic footguns):** no `datetime.now()`, `random`, or direct I/O in workflow code — use `workflow.now()` and activities; never change activity signatures once workflows exist in prod (versioning per official Temporal docs).

**Tests:** P2-T5 suite unchanged (the gate) + `tests/integration/test_temporal_durability.py::test_worker_kill_resumes` (automatable with the time-skipping test server — but mt-08 step 1 is the real `kill -9` on staging), `test_idempotent_relaunch_same_simulation_id`, `test_halt_budget_signal_mid_run`, `test_retry_policy_exhausts_at_5`.

**Edge cases:** activity succeeded but response lost (worker died after MiroShark accepted the run) → idempotency key prevents a duplicate simulation; signal arrives before `run()` starts → Temporal buffers it, verify; worker down > activity timeout → retry, not a new run; ensemble state machine must live in workflow history, not in-process memory — that's the whole point.

### P8-T2 — Production compose + Helm skeleton
`infra/docker-compose.prod.yml`: no published dev ports, TLS via reverse proxy (Caddy or Traefik — pick Caddy), resource limits on every service, restart policies, log rotation. Plus `infra/helm/` minimal chart (Deployments + Services + Ingress + PVCs for postgres/neo4j/minio; forks included). Single-command deploy documented in `infra/DEPLOY.md`.
**Done when:** mt-08 step 2 — fresh VM, one command, full stack up.

**Files:**
- `infra/docker-compose.prod.yml` — overlays the dev compose; only Caddy publishes 80/443; every service gets `deploy.resources.limits` (memory + cpus), `restart: unless-stopped`, `logging: json-file` with `max-size: 50m, max-file: 5`, and a `healthcheck`.
- `infra/Caddyfile` — automatic TLS for the production domain; security headers; `/metrics` NOT proxied (internal only, see P8-T4).
- `infra/helm/` — `Chart.yaml`, `values.yaml`, `templates/{deployment,service,ingress,pvc}.yaml` per service. Forks (bettafish, miroshark) built from `forks/*/Dockerfile*` and pushed to a registry; the chart references image tags, never `latest`.
- `infra/DEPLOY.md` — the single command (`docker compose -f infra/docker-compose.prod.yml up -d` after env injection) + DNS/TLS prerequisites + the mt-08 step 4 token-rotation procedure.

**Checks:** `docker compose -f infra/docker-compose.prod.yml config` clean in CI; a `tests/infra/test_no_published_dev_ports.py` that parses the prod compose and asserts only 80/443 published, every service has limits + restart policy.

**Edge cases:** fork image builds need their build context paths right in prod compose (relative to repo root); MinIO/postgres/neo4j data on named volumes with explicit names (predictable for the backup script); first-boot TLS issuance needs port 80 reachable — document in DEPLOY.md or the human will hit it on the fresh VM.

### P8-T3 — Secrets & config hardening
No secrets in `.env` files on disk in prod: env injection documented (Doppler/Vault/cloud secret manager — document one, Doppler). `.env.example` audited: zero real-looking values. All internal tokens rotated. CORS locked to the production domain.
**Done when:** mt-08 steps 3–4.

**Files:**
- `infra/DEPLOY.md` — the Doppler section: `doppler run -- docker compose ...`, token scoping per environment, and the `INTERNAL_SERVICE_TOKEN` rotation runbook (mt-08 step 4 must succeed by following only this).
- `.env.example` — every value replaced with an obvious placeholder (`change-me-...`); add a comment header stating real values come from Doppler.
- `scripts/check_no_secrets.sh` — CI step: greps repo for real-looking keys (`sk-`, `AKIA`, `-----BEGIN`, high-entropy hex) excluding test fixtures; fails the build.
- core-api config — `ALLOWED_ORIGINS=https://app.<prod-domain>` exactly; no `*`; reject credentials with wildcard.

**Tests:** `tests/infra/test_env_example_placeholders.py` (no value parses as a plausible secret), rotation drill is manual (mt-08 step 4).

**Edge cases:** `.env` still used in dev — `.gitignore` already covers it, verify; LiteLLM virtual keys are runtime state, not env — confirm they're not in any file; rotating `INTERNAL_SERVICE_TOKEN` requires adapters + core-api restart together — DEPLOY.md must say so (staggered restarts = 401 storm).

### P8-T4 — Observability
- Metrics: Prometheus client in core-api (`/metrics`): request rates, ensemble durations, LLM spend/hour, collector success rates. Grafana dashboard JSON committed at `infra/grafana/crowdlens.json`.
- Alerts (Grafana or Alertmanager): worker down, LLM spend > $50/h, collector failure rate > 20%, disk > 80%.
- Langfuse: project per environment; the claim-evidence eval scores visible on a dashboard.
**Done when:** mt-08 step 5 — induce a failure (stop bettafish) and watch it alert.

**Files:**
- `app/metrics.py` — `prometheus_client` (official docs): `crowdlens_http_requests_total` (method, route, status), `crowdlens_ensemble_duration_seconds` (histogram), `crowdlens_llm_spend_usd_total` (counter, labels: `model_alias` only — **not** project_id, cardinality), `crowdlens_collector_outcomes_total` (source, outcome). `/metrics` exposed on the internal interface only (Caddy does not proxy it).
- `infra/prometheus/prometheus.yml`, `infra/alertmanager/rules.yml` — the four alerts: `up{job="worker"} == 0` for 5m; `rate(crowdlens_llm_spend_usd_total[1h]) > 50`; collector failure ratio > 0.2 over 15m; node disk > 80%.
- `infra/grafana/crowdlens.json` — committed dashboard (provisioned, not hand-built in the UI).
- Langfuse: `LANGFUSE_PROJECT` per env in Doppler; the P5-T2 claim-evidence eval scores already flow there — this task only adds the dashboard view.

**Tests:** `tests/core/test_metrics.py::test_metrics_endpoint_has_custom_series`, alert rules validated with `promtool check rules` in CI.

**Edge cases:** spend metric read from `cost_ledger` deltas (source of truth internal record) — reconcile with LiteLLM spend nightly, drift >1% alerts (same rule as Lago reconciliation in platform-services); alert noise on deploy (worker down during restart) → 5m `for:` window absorbs it; label cardinality: never put `project_id`, `item_id`, or URLs on hot metrics.

### P8-T5 — Backups & DR
Nightly: `pg_dump` (crowdlens), Neo4j dump, MinIO mirror — to an off-box location (document S3 target). Restore script `infra/restore.sh` + drill instructions. Retention: 30 days.
**Done when:** mt-08 step 6 — the human actually restores into a scratch environment and runs the health endpoint.

**Files:**
- `infra/backup.sh` — cron nightly: `pg_dump` (consistent snapshot by design); `neo4j-admin database dump` (**requires the neo4j database stopped or an offline window** — schedule at the nightly low-traffic hour and document the brief unavailability; online backup is an Enterprise feature); `mc mirror` (MinIO client) of the buckets to the S3 target. Timestamped layout `s3://<bucket>/crowdlens/<date>/...`; retention via S3 lifecycle rule (30 days) — documented, not hand-deleted.
- `infra/restore.sh` — full reverse path into a scratch stack: create empty volumes → restore pg → restore neo4j dump → mirror MinIO back → `docker compose up` → `curl /api/v1/health`.
- `infra/DRILL.md` — the step-by-step for mt-08 step 6, including expected durations.

**Edge cases:** backup host must not be the prod host (off-box is the point); pg_dump of a live DB is fine, but neo4j's is not — don't pretend otherwise in the docs; verify backups are not world-readable (S3 bucket policy in DRILL.md); a backup that never restored is not a backup — the drill is the done-criterion, not the script.

### P8-T6 — Security pass
- Rate limiting on all public endpoints (slowapi): 100 req/min/user default, 10/min on `/ask`.
- RBAC re-audit: every endpoint has a role check (automated test that walks the route table).
- Dependency audit: `pip-audit` + `npm audit` clean or documented exceptions.
- AGPL compliance page: `/legal/licenses` in the UI listing the forks, linking to their source.
**Done when:** mt-08 step 7.

**Files:**
- `app/middleware/ratelimit.py` — slowapi `Limiter` (official docs); key = authenticated user id, falling back to IP; behind Caddy, trust `X-Forwarded-For` only from the proxy (configure the trusted proxy list — naive `request.client.host` behind a proxy rates the proxy, not the user). Defaults: `100/minute`; `/ask` endpoints: `10/minute`.
- `tests/security/test_rbac_route_table.py::test_every_route_has_a_role_check` — walks FastAPI `app.routes`, asserts each has an RBAC dependency (or is on an explicit allowlist: `/health`, `/metrics`, `/share/{token}`); a new unprotected route fails CI.
- `docs/SECURITY-EXCEPTIONS.md` — any pip-audit/npm-audit finding not fixable this week: CVE, affected dep, why it's tolerable, revisit date.
- `frontend/src/routes/legal/LicensesPage.tsx` — `/legal/licenses`: BettaFish + MiroShark, AGPL-3.0, links to the fork repos and upstream sources.

**Note (inconsistency to handle, not hide):** slowapi's default 429 body does not match the api-surface error envelope, and the api-surface code list has no `RATE_LIMITED` code (the adapter contract does). Add the custom slowapi handler emitting `{ "error": { "code": "VALIDATION", ... } }` is wrong — instead return `429` with envelope code... the contract needs a code. **Stop-gap without editing the contract:** use code `INTERNAL`? No — never lie. Use the envelope shape with code `RATE_LIMITED` and flag the missing code to the human for a contract bump; do not silently pick a code.

**Tests:** `test_rate_limit_ask_10_per_min` (burst 150 on a default route → 429s per mt-08 step 7), `test_forwarded_for_only_from_proxy`, `test_rbac_route_table`, audits run in CI (`pip-audit --strict`, `npm audit --audit-level=high`).

**Edge cases:** WS connections are not HTTP-rate-limited — limit connection opens instead; shared office IPs (many users one NAT) → user-id keying mostly avoids it, IP fallback is the leak; health checks must never be rate-limited (load balancer dies otherwise).

### P8-T7 — Enterprise: SSO + audit export + on-prem profile
- AUTH-5: SSO via Logto enterprise connectors (SAML/OIDC) — per-workspace toggle in `/admin`.
- AUTH-7: audit log export (CSV) + 90-day hot retention note.
- AUTH-8/F-21: on-prem profile = the phase-0 compose + a `profiles/onprem` overlay: LiteLLM pointed at local vLLM/Ollama endpoints (config-only change — document GPU tiers 24GB/48GB/2×80GB per PBR §11), license-key check at startup (signed JWT license file, offline grace 30 days).
**Done when:** mt-08 steps 8–9.

**Files:**
- `services/core-api/alembic/versions/00XX_enterprise.py` — `sso_configs` (+ `workspace_branding` if not already migrated in phase 5), DDL verbatim from data-model.md.
- `app/routers/admin.py` — SSO toggle endpoints (owner only); Logto connector creation/config via the Logto Management API (official Logto docs — do not hand-roll SAML).
- `GET /workspaces/{id}/audit/export?format=csv` — streams `audit_events` for the workspace; **add to the api-surface.md index first (R1)**; the 90-day hot retention is a note in `/admin` + `DEPLOY.md` (older rows archived to MinIO, not deleted — state this).
- `profiles/onprem/docker-compose.override.yml` + `profiles/onprem/litellm.config.yaml` — model aliases (`swarm-model`, `report-model`, `embed-model`) re-pointed at `http://ollama:11434` / vLLM endpoints; GPU tier guide in `profiles/onprem/README.md` (24GB / 48GB / 2×80GB per PBR §11).
- `app/services/license.py` — startup check.

**License check (the no-improvised-crypto zone — stop condition):** license = a JWT signed RS256 by CrowdLens; core-api embeds only the public key; verify with PyJWT per its official docs (`jwt.decode(token, PUBLIC_KEY, algorithms=["RS256"])` — pin `algorithms`, never accept `none` or HS256 confusion). Claims: `workspace_name`, `exp`, `seats`. Grace: if the file is expired by ≤30 days → run with a loud warning banner in `/admin`; >30 days or missing/tampered → clean refusal at startup (mt-08 step 9). Path from env `CROWDLENS_LICENSE_PATH`.

**Tests:** `test_license_valid_starts`, `test_license_expired_within_grace_warns`, `test_license_expired_beyond_grace_refuses`, `test_license_tampered_signature_refuses`, `test_hs256_alg_confusion_rejected`, `test_sso_toggle_owner_only`, `test_audit_export_csv_wellformed`, plus mt-08 step 9's zero-external-LLM-calls check (assert LiteLLM config has no upstream provider base URLs when the overlay is active).

**Edge cases:** on-prem machine clock skew → JWT `exp` check with a documented leeway; SSO login for a user not yet in `users` → provision on first login with the IdP-mapped role (audit action `sso_login` already exists in the `audit_events` action list); offline grace requires the license file readable at every startup, not just install day.

### P8-T8 — Load test + launch checklist
`tests/load/locustfile.py`: 50 concurrent viewers on theater WS + 10 concurrent decision cycles. Targets: p95 API < 500ms, WS event latency < 2s, no 5xx. Then `infra/LAUNCH-CHECKLIST.md`: every gate from mt-00→mt-08 re-run against production.
**Done when:** mt-08 step 10 + the launch checklist fully ticked.

**Files:**
- `tests/load/locustfile.py` — two user classes:
  - `TheaterViewerUser` (weight 50) — Locust has no built-in WS client; write a custom client per the official Locust docs (events fired with `request_type="WS"`), connecting via `POST /simulations/{sid}/ws-ticket` → `WS /ws/simulations/{sid}`, measuring event latency from server timestamp to receipt.
  - `DecisionCycleUser` (weight 10) — full cycle against staging with **small, capped ensembles** (spend caps set tight on the load-test projects; a load test must not burn real LLM budget — use the lowest-cost config or a mocked-adapter staging profile).
- `infra/LAUNCH-CHECKLIST.md` — a literal checklist: every mt-00→mt-08 step, re-run against the production/staging environment, sign-off column. This file is the launch evidence pack (enterprise buyers ask for mt-08 steps 6–7 — say so in the header, matching mt-08's sign-off note).

**Targets (asserted in the locust run, not eyeballed):** p95 API < 500ms; WS event latency < 2s; zero 5xx. Miss → stop condition below, no silent relaxation.

**Edge cases:** WS reconnect storms (kill a viewer's connection mid-test — the client must reconnect without doubling load); ensemble runs stampeding LiteLLM rate limits during the test → that's real data, report it; run the test against staging, never prod data; tear down load-test workspaces after (they pollute `cost_ledger` reconciliation).

## Stop conditions
- Temporal migration changes ensemble behavior → that's a bug, not a refactor detail. Roll back, report.
- Load targets missed → report numbers; do not relax targets silently.
- License-key scheme is the one place you may NOT improvise cryptography — use signed JWTs (RS256) exactly as specified, or stop.

## Manual test
`../manual-tests/mt-08-production.md` — the final gate.
