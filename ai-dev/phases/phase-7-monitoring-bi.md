# Phase 7 — Monitoring, Alerts & BI Layer

**Goal:** the product keeps watching after the verdict — scheduled collection, shift alerts, dashboards, narrative tracking, public API. **Gate:** `manual-tests/mt-07-monitoring-bi.md` green.

## Prerequisites
- Phase 6 green. Novu + Lago added to compose (human adds their API keys to `.env`). Read `contracts/platform-services.md` first.

## Tasks

### P7-T1 — Scheduled collectors (monitoring jobs)
Create `monitoring_schedules` **per `contracts/data-model.md` → Later-phase tables**. A scheduler (APScheduler in core-api for now; Temporal in phase 8) launches grounding jobs per schedule and diffs against the previous run.
**Done when:** integration test with a 1/min schedule on a mocked adapter: two cycles run, items stored, second cycle links to first.

### P7-T2 — Sentiment-shift detection + alerts
`app/services/shift_detect.py`: per project, per theme — z-score vs trailing 7-run baseline; |z| > 2.5 → shift event with the top driver items. Fires Novu trigger `sentiment_shift` (per platform-services contract). **Never alert without driver items** — "sentiment dropped" without the thread that caused it is noise.
**Done when:** unit tests on synthetic series (spike detected, flat series silent); mt-07 step 3.

### P7-T3 — Narrative tracking (F-39) + anomaly detection (F-40)
`app/services/narratives.py`: cluster collected items per monitoring run (embedding + HDBSCAN via `report-model` embeddings), match clusters to prior narratives (cosine ≥ 0.8), track lifecycle stage (emerging/peaking/declining by volume slope) + momentum score. Anomaly: volume z-score per narrative. Stored in `narratives` + `narrative_snapshots` **per `contracts/data-model.md` → Later-phase tables**.
**Done when:** unit tests on fixture time-series; narratives endpoint `GET /projects/{id}/narratives` serves the data.

### P7-T4 — Dashboards (BI v1)
`/dashboards`: KPI cards + charts per project/workspace — sentiment index, share-of-voice, issue volume, narrative momentum (from P7-T3), per-market and per-language cuts (phase-6 language data). ECharts, TanStack Query, evidence drawer on every data point (frontend-spec rule 1).
**Done when:** mt-07 step 5.

### P7-T5 — Weekly digest (F-41 lite)
Monday 09:00 workspace-local: per active project, `report-model` writes "what changed / why / what to watch" from the week's monitoring diffs; every claim cited; delivered via Novu `weekly_digest`.
**Done when:** mt-07 step 6 — trigger manually, email arrives, claims cited.

### P7-T6 — Public API + webhooks (F-20, AUTH-6)
Scoped service tokens (`cpub_...`, hashed at rest, scopes: `read:projects`, `read:verdicts`, `write:grounding`...): `POST /workspaces/{id}/api-keys` (owner). Versioned public surface = read-only subset of api-surface + `verdicts` + `narratives`. Webhook management endpoints + signed delivery per platform-services §webhooks, with delivery log + replay.
**Done when:** integration tests: scope enforcement (a `read:verdicts` token gets 403 on grounding), signature verification, replay; mt-07 step 7.

### P7-T7 — Validation Center (F-18 v1)
`/validation`: decision registry table UI (verdict, confidence, outcome when logged, hit/miss), outcome-entry form (analyst logs what really happened), accuracy summary (theme-level hit rate with n — hide the % when n < 10: no fake precision).
**Done when:** mt-07 step 8.

## Stop conditions
- Novu/Lago self-host setup diverges from docs → stop, report the exact mismatch.
- Narrative clustering is garbage on real data (mt-07 step 4 eyeball) → tune once; still bad → stop and report. Don't ship decorative clusters.

## Manual test
`../manual-tests/mt-07-monitoring-bi.md`
