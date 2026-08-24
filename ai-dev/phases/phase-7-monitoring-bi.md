# Phase 7 — Monitoring, Alerts & BI Layer

**Goal:** the product keeps watching after the verdict — scheduled collection, shift alerts, dashboards, narrative tracking, public API. **Gate:** `manual-tests/mt-07-monitoring-bi.md` green.

## Prerequisites
- Phase 6 green. Novu + Lago added to compose (human adds their API keys to `.env`). Read `contracts/platform-services.md` first.

## Tasks

### P7-T1 — Scheduled collectors (monitoring jobs)
Create `monitoring_schedules` **per `contracts/data-model.md` → Later-phase tables**. A scheduler (APScheduler in core-api for now; Temporal in phase 8) launches grounding jobs per schedule and diffs against the previous run.
**Done when:** integration test with a 1/min schedule on a mocked adapter: two cycles run, items stored, second cycle links to first.

**Files:**
- `services/core-api/alembic/versions/00XX_monitoring.py` — creates `monitoring_schedules` (plus `narratives`, `narrative_snapshots`, `outcome_logs`, `service_tokens`, `webhook_endpoints`, `webhook_deliveries` — copy the DDL verbatim from data-model.md, never retype it).
- `app/services/scheduler.py` — APScheduler `AsyncIOScheduler` (official APScheduler docs; pinned in requirements). Loads `active = true` schedules at startup; a DB-change listener (or 30s poll) picks up edits without restart.
- `app/services/monitoring.py` — run orchestration + diff.
- `app/routers/monitoring.py` — schedule CRUD (already indexed in api-surface.md: "monitoring schedule CRUD").
- `tests/integration/test_monitoring_schedule.py`.

**Key functions:**
```python
def register_schedules(scheduler: AsyncIOScheduler) -> None
async def run_schedule(schedule_id: UUID, trigger: Literal["cron", "run_now"]) -> UUID  # new job_id
def diff_runs(prev_job_id: UUID, new_job_id: UUID) -> RunDiff  # {added: [item_id], removed: [...], counts}
```
Cron strings validated at write time (reject garbage with `VALIDATION`, don't let the scheduler discover it at 3am). `last_run_at` updated only on successful job creation. "Second cycle links to first": the diff is computed between the project's two most recent grounding jobs; the link is `diff.prev_job_id` — no schema change needed. On completion of the re-simulation, emit Lago `monitoring_run` (`external_customer_id = workspace_id`, property `project_id`) per platform-services §Lago.

**Tests:** `test_two_cycles_link` (the done-criterion), `test_invalid_cron_rejected`, `test_inactive_schedule_never_fires`, `test_manual_run_now` (mt-07 step 2's trigger), `test_diff_empty_when_no_change`, `test_adapter_failure_marks_job_failed_not_silent`.

**Edge cases:** overlapping runs (cron fires while the previous run is still collecting → skip + log, never stack duplicate jobs); schedule created for a project with no prior run → first run diffs against nothing (diff = all-added, flagged `baseline: true`); deleted project → `ON DELETE CASCADE` removes the schedule, confirm the scheduler unregisters the job; server restart mid-window → missed cron fires once on recovery (misfire grace), not in a loop.

### P7-T2 — Sentiment-shift detection + alerts
`app/services/shift_detect.py`: per project, per theme — z-score vs trailing 7-run baseline; |z| > 2.5 → shift event with the top driver items. Fires Novu trigger `sentiment_shift` (per platform-services contract). **Never alert without driver items** — "sentiment dropped" without the thread that caused it is noise.
**Done when:** unit tests on synthetic series (spike detected, flat series silent); mt-07 step 3.

**Files:**
- `app/services/shift_detect.py` — pure statistics, stdlib `statistics` only. No LLM anywhere in this file.
- `app/services/notify.py` — thin Novu wrapper (trigger ids from platform-services §Novu, never string literals scattered in callers).
- `tests/core/test_shift_detect.py`.

**Key functions:**
```python
def zscore(value: float, baseline: list[float]) -> float          # sample stdev; stdev==0 → 0 unless value≠mean
def detect_shifts(project_id: UUID) -> list[ShiftEvent]           # per theme, vs trailing 7 runs
def driver_items(theme: str, job_id: UUID, limit: int = 3) -> list[str]  # top items by engagement in the theme
async def fire_shift_alert(event: ShiftEvent) -> None             # Novu `sentiment_shift`
```
`ShiftEvent = {project_id, theme, z, direction, driver_item_ids}` — the event is refused (returned as `None`, logged) if `driver_item_ids` is empty. Alert copy is a fixed template with the theme name + deep links to driver items (frontend rule: every notification deep-links) — no LLM-generated prose, so it cannot invent a cause. Dedupe: same (project, theme, direction) within 48h → suppressed.

**Tests:** `test_spike_detected` (|z| = 3 series), `test_flat_series_silent`, `test_driverless_anomaly_produces_no_alert` (mt-07 step 3's silence check), `test_fewer_than_7_runs_no_baseline_no_alert`, `test_zero_stdev_flat_baseline`, `test_dedupe_within_48h`, `test_low_volume_theme_skipped` (< 5 items in the theme this run → skip; small-n z-scores are noise).

**Edge cases:** exactly-7-runs boundary (baseline = runs 1–7, evaluate run 8); first-ever monitoring run → no baseline → no alert, ever; theme disappears entirely (volume 0) → that's a lifecycle event for P7-T3, not a shift alert; Novu down → log + retry via background task, the shift event row is persisted regardless so the alert can be re-fired.

### P7-T3 — Narrative tracking (F-39) + anomaly detection (F-40)
`app/services/narratives.py`: cluster collected items per monitoring run (embedding + HDBSCAN via `report-model` embeddings), match clusters to prior narratives (cosine ≥ 0.8), track lifecycle stage (emerging/peaking/declining by volume slope) + momentum score. Anomaly: volume z-score per narrative. Stored in `narratives` + `narrative_snapshots` **per `contracts/data-model.md` → Later-phase tables**.
**Done when:** unit tests on fixture time-series; narratives endpoint `GET /projects/{id}/narratives` serves the data.

Correction on embeddings: use the **`embed-model` alias** (the same one phase 5's `report_embeddings` uses — 1024-dim), not `report-model`; `report-model` is a generation alias.

**Files:**
- `app/services/narratives.py` — clustering, matching, lifecycle.
- `app/routers/narratives.py` — `GET /projects/{id}/narratives` (already in the api-surface index).
- `tests/core/test_narratives.py` + `tests/core/fixtures/narrative_series.json` — fixture time-series of pre-embedded items.
- Dependency: `hdbscan` (pin; API per official docs — `HDBSCAN(min_cluster_size=...)` → `.fit(vectors)` → `.labels_`).

**Key functions:**
```python
async def embed_items(item_ids: list[str]) -> list[list[float]]        # embed-model via LiteLLM, batched
def cluster(vectors: list[list[float]]) -> list[int]                   # HDBSCAN labels; -1 = noise
def match_to_prior(centroid: list[float], priors: list[Narrative]) -> UUID | None  # cosine ≥ 0.8
def classify_lifecycle(volumes: list[int]) -> Literal["emerging","peaking","declining"]
def momentum(volumes: list[int]) -> float                              # slope of last 3 snapshots, normalized
async def snapshot_run(project_id: UUID, job_id: UUID) -> int          # rows written
```
Cluster label text: derived from the cluster's top driver items (`report-model` may phrase the label, but `driver_item_ids` must be real item ids from the cluster — a label with zero driver ids is rejected). Noise points (label `-1`) never become a narrative. A narrative with no matching cluster for 3 consecutive runs → keep history, stop writing snapshots (it's in `declining`, then dormant — don't delete).

**Tests:** `test_fixture_clusters_sensible` (fixture with 3 planted themes → 3 clusters), `test_noise_not_narrativized`, `test_match_threshold_0_8` (0.79 → new narrative, 0.81 → matched), `test_lifecycle_classification` (rising/flat/falling fixture series), `test_label_without_driver_ids_rejected`, `test_too_few_items_skips_run` (< `min_cluster_size` items → no rows, no error).

**Edge cases:** embedding dim drift (embed-model swap → vectors incompatible with stored centroids → detect dim mismatch, log, rebuild centroids from scratch rather than crash); single-item run; all items identical (duplicate forwards) → one giant cluster is correct, not a bug; HDBSCAN non-determinism across versions → pinned version + fixture test catches it.

**Cheap-LLM failure modes:** weak embeddings → garbage clusters (the mt-07 step 4 eyeball is the gate — stop condition below); `report-model` inventing a dramatic label for a mundane cluster → label is display-only, every number on screen comes from `narrative_snapshots`, never from the label text.

### P7-T4 — Dashboards (BI v1)
`/dashboards`: KPI cards + charts per project/workspace — sentiment index, share-of-voice, issue volume, narrative momentum (from P7-T3), per-market and per-language cuts (phase-6 language data). ECharts, TanStack Query, evidence drawer on every data point (frontend-spec rule 1).
**Done when:** mt-07 step 5.

**Files:**
- `frontend/src/routes/dashboards/DashboardPage.tsx` — route, project/workspace switcher.
- `frontend/src/components/` — `SentimentIndexChart.tsx`, `ShareOfVoiceChart.tsx`, `IssueVolumeChart.tsx`, `NarrativeMomentumChart.tsx`, `KpiCard.tsx` (domain-named per frontend-spec conventions).
- `frontend/src/hooks/useDashboard.ts` — TanStack Query only (frontend rule 6).
- Backend: aggregates need server-side SQL (per-source sentiment over time, per-language cuts). If the existing items/narratives endpoints can't serve a cut without shipping raw items to the client, add e.g. `GET /projects/{id}/dashboard/summary` — **to the api-surface.md extended index first (R1)**, then implement in `app/routers/dashboard.py`.
- `tests/frontend/` — one vitest file per component.

**Rendering rules:** every chart point's click → `EvidenceDrawer` with the underlying items (real posts, platform+date+link); narrative momentum chart links to the narratives view. Empty project → honest empty state ("no monitoring runs yet"), never zeros drawn as data. Per-language cut uses the `language` column incl. `hi-Latn`; `und` bucket shown as "undetermined".

**Tests:** `KpiCard.test.tsx::renders_evidence_link`, `DashboardPage.test.tsx::empty_state_no_fake_zeros`, plus mt-07 step 5 by hand.

**Edge cases:** single monitoring run → trends render as a point, not a fake flat line; language cut with one language → hide the cut control rather than show a 100% bar (no decimal-point theater); dashboard of a workspace with 20 projects → summary endpoint must aggregate in SQL, not N+1 per project.

### P7-T5 — Weekly digest (F-41 lite)
Monday 09:00 workspace-local: per active project, `report-model` writes "what changed / why / what to watch" from the week's monitoring diffs; every claim cited; delivered via Novu `weekly_digest`.
**Done when:** mt-07 step 6 — trigger manually, email arrives, claims cited.

**Files:**
- `app/services/digest.py` — assembly + generation + post-check.
- `app/services/notify.py` — add the `weekly_digest` trigger call.
- Scheduler entry in `app/services/scheduler.py` — weekly cron; "workspace-local" needs a timezone per workspace (see note below).
- `tests/core/test_digest.py`.

**Key functions:**
```python
def collect_week(project_id: UUID, since: datetime) -> WeekDigest      # diffs, shift events, narrative changes
async def write_digest(digest: WeekDigest) -> str                      # report-model; markdown
def check_citations(markdown: str, digest: WeekDigest) -> str          # strip any claim without a ref
```
**Timezone note:** `workspaces` has no tz column (data-model.md). Until one is added, schedule per workspace using the owner's profile timezone, defaulting to `Asia/Kolkata` — record the limitation in the code comment; do not add a column ad hoc (R1). 

**Post-check (same doctrine as P2-T6 verdicts):** every paragraph in the digest must reference a diff, shift event, or narrative id from `collect_week`; unreferenced paragraphs are removed. If >50% of paragraphs are stripped, discard and log — the model is confabulating.

**Tests:** `test_all_claims_cited` (fixture digest → zero unreferenced paragraphs), `test_no_changes_short_digest` (quiet week → "nothing material changed" email, not invented drama), `test_skips_projects_without_monitoring`, `test_confabulation_discards_digest`.

**Edge cases:** project paused mid-week; shift alert already sent this week → digest references it, doesn't re-alert; workspace with zero active projects → no email at all (not an empty email).

**Cheap-LLM failure modes:** invented numbers (numeral guard: every number in the output must appear in `collect_week`'s data); "what to watch" generic filler → must tie to a narrative/shift id or be cut; translated quotes (phase-6 rule) → quotes stay original-script in the email too.

### P7-T6 — Public API + webhooks (F-20, AUTH-6)
Scoped service tokens (`cpub_...`, hashed at rest, scopes: `read:projects`, `read:verdicts`, `write:grounding`...): `POST /workspaces/{id}/api-keys` (owner). Versioned public surface = read-only subset of api-surface + `verdicts` + `narratives`. Webhook management endpoints + signed delivery per platform-services §webhooks, with delivery log + replay.
**Done when:** integration tests: scope enforcement (a `read:verdicts` token gets 403 on grounding), signature verification, replay; mt-07 step 7.

**Files:**
- `app/services/api_keys.py` — token lifecycle.
- `app/services/webhooks.py` — signing + delivery + retry.
- `app/dependencies.py` — `require_scope(scope: str)` FastAPI dependency for the public surface; service-token auth is separate from the Logto JWT path (a `cpub_` token is never a user).
- `app/routers/public_api.py` (versioned `/api/v1/public/...` read-only subset) and `app/routers/webhooks.py` (management).
- `tests/integration/test_public_api.py`, `tests/integration/test_webhooks.py`.

**Key functions:**
```python
def create_token(workspace_id: UUID, name: str, scopes: list[str]) -> str  # "cpub_"+secrets.token_urlsafe(32); shown ONCE; sha256 stored
def verify_token(token: str) -> TokenContext | None                        # hash lookup; revoked_at → None
def sign(secret: str, body: bytes) -> str                                  # HMAC-SHA256, hex
async def deliver(delivery_id: int) -> None                                # at-least-once; 5 retries over 1h
async def replay(delivery_id: int) -> None                                 # re-send identical payload + signature
```
Tables already fixed in data-model.md (`service_tokens`, `webhook_endpoints`, `webhook_deliveries`) — migrate verbatim. Event names and payload key fields come from platform-services §webhooks exactly (`grounding.completed`, `simulation.completed`, `verdict.issued`, `budget.halted`). Header: `X-CrowdLens-Signature: sha256=<hex>`. Every metered public request also emits Lago `api_call` (property `endpoint`) on metered tiers. Replay sends the stored payload byte-identical — signature still validates (mt-07 step 7).

**Tests:** `test_read_verdicts_token_403_on_grounding` (the done-criterion), `test_token_shown_once`, `test_revoked_token_401`, `test_unknown_scope_rejected_at_creation`, `test_signature_validates` (recompute HMAC in the test), `test_wrong_secret_fails`, `test_replay_identical_payload`, `test_delivery_retry_backoff` (mocked 500s → 5 attempts, then `failed`).

**Edge cases:** endpoint URL returning 301 → follow once, log; endpoint down through all retries → `status='failed'`, surfaced in the deliveries list (no silent loss); token with `write:grounding` still cannot touch another workspace (workspace scoping is checked before scope); delivery id is the idempotency handle receivers use — document that in the endpoint description.

### P7-T7 — Validation Center (F-18 v1)
`/validation`: decision registry table UI (verdict, confidence, outcome when logged, hit/miss), outcome-entry form (analyst logs what really happened), accuracy summary (theme-level hit rate with n — hide the % when n < 10: no fake precision).
**Done when:** mt-07 step 8.

**Files:**
- Backend: outcome logging + accuracy endpoints are new — **add them to the api-surface.md index first (R1)**: e.g. `POST /verdicts/{id}/outcome` (analyst+), `GET /workspaces/{id}/validation`, `GET /workspaces/{id}/validation/accuracy`.
- `app/services/validation.py` — `log_outcome(verdict_id, outcome, notes, user)`, `accuracy(workspace_id) -> per-theme {hits, partials, misses, n}`.
- `frontend/src/routes/validation/ValidationPage.tsx`, `OutcomeForm.tsx`, `AccuracySummary.tsx` (domain components).
- `tests/core/test_validation.py`, `tests/frontend/AccuracySummary.test.tsx`.

**Rules:** `outcome_logs` is append-only (data-model invariant 7) — a corrected outcome is a new row; the accuracy read takes the latest row per verdict. Hit-rate definition, stated in the UI label so nobody argues about it: `hits / (hits + partials + misses)`, partials count in n but not as hits. `n < 10` → render the count table only, no `%` (mt-07 step 8 expects exactly this).

**Tests:** `test_outcome_append_only` (UPDATE attempt rejected), `test_accuracy_hidden_below_10`, `test_latest_outcome_wins`, `test_partial_counts_in_n_not_as_hit`.

**Edge cases:** verdict with `no_consensus` outcome → not in the registry (nothing to be right about); outcome logged against an archived project → allowed (reality doesn't wait); theme derived from the verdict's dominant objection theme — if ambiguous, bucket "untagged" rather than guess.

## Stop conditions
- Novu/Lago self-host setup diverges from docs → stop, report the exact mismatch.
- Narrative clustering is garbage on real data (mt-07 step 4 eyeball) → tune once; still bad → stop and report. Don't ship decorative clusters.

## Manual test
`../manual-tests/mt-07-monitoring-bi.md`
