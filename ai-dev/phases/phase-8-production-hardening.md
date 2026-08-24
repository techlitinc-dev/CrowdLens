# Phase 8 — Production Hardening & Enterprise

**Goal:** from "works on my machine" to deployable product: Temporal orchestration, prod infra, observability, security, SSO, on-prem profile. **Gate:** `manual-tests/mt-08-production.md` green — this is the launch checklist.

## Prerequisites
- Phase 7 green. Human decisions needed upfront: production domain, cloud host, SMTP/WhatsApp providers for Novu, Lago plans created.

## Tasks

### P8-T1 — Temporal migration
Move `ensemble` orchestration, grounding polling, monitoring schedules, and export jobs onto Temporal per `contracts/platform-services.md` §Temporal. Same business logic, durable execution. The adaptive-ensemble integration tests from P2-T5 must pass unchanged against the Temporal implementation (behavior-preserving refactor only — R6).
**Done when:** kill -9 the worker mid-ensemble → `docker compose start worker` → ensemble resumes from its last checkpoint and completes; P2-T5 tests green.

### P8-T2 — Production compose + Helm skeleton
`infra/docker-compose.prod.yml`: no published dev ports, TLS via reverse proxy (Caddy or Traefik — pick Caddy), resource limits on every service, restart policies, log rotation. Plus `infra/helm/` minimal chart (Deployments + Services + Ingress + PVCs for postgres/neo4j/minio; forks included). Single-command deploy documented in `infra/DEPLOY.md`.
**Done when:** mt-08 step 2 — fresh VM, one command, full stack up.

### P8-T3 — Secrets & config hardening
No secrets in `.env` files on disk in prod: env injection documented (Doppler/Vault/cloud secret manager — document one, Doppler). `.env.example` audited: zero real-looking values. All internal tokens rotated. CORS locked to the production domain.
**Done when:** mt-08 steps 3–4.

### P8-T4 — Observability
- Metrics: Prometheus client in core-api (`/metrics`): request rates, ensemble durations, LLM spend/hour, collector success rates. Grafana dashboard JSON committed at `infra/grafana/crowdlens.json`.
- Alerts (Grafana or Alertmanager): worker down, LLM spend > $50/h, collector failure rate > 20%, disk > 80%.
- Langfuse: project per environment; the claim-evidence eval scores visible on a dashboard.
**Done when:** mt-08 step 5 — induce a failure (stop bettafish) and watch it alert.

### P8-T5 — Backups & DR
Nightly: `pg_dump` (crowdlens), Neo4j dump, MinIO mirror — to an off-box location (document S3 target). Restore script `infra/restore.sh` + drill instructions. Retention: 30 days.
**Done when:** mt-08 step 6 — the human actually restores into a scratch environment and runs the health endpoint.

### P8-T6 — Security pass
- Rate limiting on all public endpoints (slowapi): 100 req/min/user default, 10/min on `/ask`.
- RBAC re-audit: every endpoint has a role check (automated test that walks the route table).
- Dependency audit: `pip-audit` + `npm audit` clean or documented exceptions.
- AGPL compliance page: `/legal/licenses` in the UI listing the forks, linking to their source.
**Done when:** mt-08 step 7.

### P8-T7 — Enterprise: SSO + audit export + on-prem profile
- AUTH-5: SSO via Logto enterprise connectors (SAML/OIDC) — per-workspace toggle in `/admin`.
- AUTH-7: audit log export (CSV) + 90-day hot retention note.
- AUTH-8/F-21: on-prem profile = the phase-0 compose + a `profiles/onprem` overlay: LiteLLM pointed at local vLLM/Ollama endpoints (config-only change — document GPU tiers 24GB/48GB/2×80GB per PBR §11), license-key check at startup (signed JWT license file, offline grace 30 days).
**Done when:** mt-08 steps 8–9.

### P8-T8 — Load test + launch checklist
`tests/load/locustfile.py`: 50 concurrent viewers on theater WS + 10 concurrent decision cycles. Targets: p95 API < 500ms, WS event latency < 2s, no 5xx. Then `infra/LAUNCH-CHECKLIST.md`: every gate from mt-00→mt-08 re-run against production.
**Done when:** mt-08 step 10 + the launch checklist fully ticked.

## Stop conditions
- Temporal migration changes ensemble behavior → that's a bug, not a refactor detail. Roll back, report.
- Load targets missed → report numbers; do not relax targets silently.
- License-key scheme is the one place you may NOT improvise cryptography — use signed JWTs (RS256) exactly as specified, or stop.

## Manual test
`../manual-tests/mt-08-production.md` — the final gate.
