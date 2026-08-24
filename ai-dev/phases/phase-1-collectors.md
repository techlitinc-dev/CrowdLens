# Phase 1 — Grounding Collectors

**Goal:** real Reddit + YouTube data flows from public APIs through the BettaFish adapter into `collected_items`, citation-verified analysis included. **Gate:** `manual-tests/mt-01-collectors.md` all green.

## Prerequisites
- Phase 0 complete and green
- Human adds to `.env`: `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`, `REDDIT_USER_AGENT`, `YOUTUBE_API_KEY`
- BettaFish fork cloned into `forks/bettafish` (human runs `git clone <fork-url>`; agent does not modify fork code — R2)

## Tasks

### P1-T1 — BettaFish service in compose
Add `bettafish` service to `infra/docker-compose.yml`: builds from `../forks/bettafish`, port `5001`, env `INTERNAL_SERVICE_TOKEN`, network `crowdlens-net`, depends_on postgres+redis. If the fork lacks a Dockerfile, write one in `forks/bettafish/Dockerfile.adapter` — **the only file you may add inside a fork**; do not touch its code.
**Done when:** `docker compose up -d bettafish` healthy; its `/health` (or root) responds on `localhost:5001`.

**Files:**
- `infra/docker-compose.yml` — append one service: `build: context: ../forks/bettafish` (`dockerfile: Dockerfile.adapter` if that's the one in use), `ports: ["5001:<fork-port>"]`, env `INTERNAL_SERVICE_TOKEN` plus the fork's own required env passed through from `.env`, `networks: [crowdlens-net]`, `depends_on: postgres: service_healthy, redis: service_healthy`, and a healthcheck against whichever path the fork actually answers (`/` or `/health` — read the fork's README/code to find out, don't guess).
- `forks/bettafish/Dockerfile.adapter` — only if the fork ships no Dockerfile. Base image per the fork's own requirements (its `requirements.txt`/`pyproject.toml` dictates the Python version); the container's start command is whatever the fork's README documents. R2: this Dockerfile is the only permitted addition inside the fork.

**Tests:** none automated — mt-01 step 1 is the test.

**Edge cases:**
- The fork may listen on a port other than 5001 internally — map host 5001 to the fork's real port rather than reconfiguring the fork.
- The fork may require its own env vars (DB DSN, redis URL) — point them at the compose service names (`postgres`, `redis`), never `localhost`.
- First build can take minutes (fork dependency install); a slow build is not a failure — a missing base image pin is.

**Failure modes (cheap LLM):**
- "Fixing" the fork when it doesn't boot — patch impulses go through the stop condition, not the fork's source (R2).
- Guessing the healthcheck path; container marked unhealthy while the service actually works, or vice versa.
- Hardcoding fork secrets into the compose file instead of env passthrough from `.env`.
- Exposing extra fork ports "for debugging" — only 5001 is published.

### P1-T2 — Adapter module: BettaFish
`services/adapters/adapters/bettafish.py`: async httpx client implementing every endpoint in `contracts/adapter-contract.md` §1 with the exact payload shapes. Includes `_strip_pii(item)` applied to every collected item: removes `author`, `username`, `avatar`, profile URLs before anything is stored. Retries: 3× exponential backoff on `retryable: true` errors only.
Unit tests with `respx`-mocked HTTP: happy path, PII stripping proof, error envelope mapping.
**Done when:** `pytest services/adapters` green; a test asserts a fixture item containing `author: "someuser"` is stored without it.

**Files:**
- `services/adapters/adapters/bettafish.py` — the whole module:
  - `class AdapterError(Exception)` — carries `code: str`, `message: str`, `retryable: bool`; raised from the adapter-contract error envelope verbatim (R10: message passed through, never rewritten).
  - `class BettaFishAdapter` — `def __init__(self, base_url: str = "http://bettafish:5001", token: str, client: httpx.AsyncClient | None = None)`. Header `X-Internal-Token: <token>` on every request.
  - `async def start_collect(self, payload: dict) -> dict` — POST `/collect`, body exactly contract §1 (job_id, topic, keywords, sources, region, languages, max_items_per_source, time_window_days).
  - `async def get_status(self, job_id: str) -> dict` — GET `/collect/{job_id}/status`.
  - `async def get_items(self, job_id: str, source: str | None = None, offset: int = 0, limit: int = 100) -> dict` — GET `/collect/{job_id}/items`; **every item in the response passes through `_strip_pii` before returning**.
  - `async def start_analyze(self, job_id: str) -> dict` — POST `/analyze/{job_id}`.
  - `async def get_analysis(self, report_id: str) -> dict` — GET `/analyze/{report_id}`.
  - `def _strip_pii(item: dict) -> dict` — drops keys `author`, `username`, `avatar`, `author_url`, `profile_url` (and any key ending in `_url` whose value matches a profile/avatar pattern) from a shallow copy; never mutates the input.
  - `async def _request(self, method, path, **kw) -> dict` — shared wrapper: parses the error envelope, retries 3× with exponential backoff (e.g. 0.5s/1s/2s) **only when `retryable: true`** or on transport-level `httpx.TransportError`; anything else raises immediately.
- `services/adapters/pyproject.toml` — add pinned `httpx`, `pytest`, `pytest-asyncio`, and `respx` (test-only) — named by this task, so R8-legal.

**Tests:** `services/adapters/tests/test_bettafish.py` (respx-mocked; fixtures in `services/adapters/tests/fixtures/`)
- `test_start_collect_happy_path` — 202 envelope passes through unchanged.
- `test_get_status_happy_path` — progress dict parsed as-is.
- `test_get_items_strips_author` — fixture item with `author: "someuser"` returns without the key (this is the done-criteria test).
- `test_strip_pii_removes_avatar_and_profile_urls` — `avatar`, `profile_url` gone; `url` (the canonical post URL) kept.
- `test_strip_pii_does_not_mutate_input`.
- `test_retryable_error_retried_three_times` — first two responses `{error:{code:"UPSTREAM_FAILURE", retryable:true}}`, third succeeds; assert exactly 3 HTTP calls.
- `test_non_retryable_error_raises_immediately` — `INVALID_INPUT, retryable:false` → `AdapterError` after exactly 1 call.
- `test_error_code_and_message_preserved` — raised `AdapterError.code == "RATE_LIMITED"`, message verbatim.
- `test_transport_error_retried_then_raised` — connection errors retry, then raise `AdapterError(code="UPSTREAM_FAILURE")`.
- `test_internal_token_header_sent` — assert `X-Internal-Token` on the recorded request.

**Edge cases:**
- Item missing optional fields (`published_at: null`, no `language`) — stripper must not crash on absent keys.
- PII nested under a sub-object (e.g. `author: {"name": ..., "avatar": ...}`) — drop the whole key, don't deep-clean.
- Error body that isn't the envelope (proxy 502 HTML page) — map to `UPSTREAM_FAILURE`, include status code in the message, don't JSON-crash.
- `retryable: true` on the 3rd failure too — stop at 3 attempts and raise; backoff is bounded.

**Failure modes (cheap LLM):**
- Retrying everything (or retrying on `INVALID_INPUT`) — the contract's `retryable` flag exists precisely to prevent hammering a request that will never succeed.
- Stripping PII at storage time in core-api instead of at the adapter boundary — then raw items leak into adapter logs/caches (R5 says boundary).
- Swallowing the envelope into `Exception("request failed")` — R10; the code/message must survive for core-api to map into its own error envelope.
- Inventing convenience methods (`get_all_items()` with hidden auto-pagination) not in contract §1 — pagination decisions belong to the caller (P1-T3).
- Mutating the response dict in `_strip_pii` and causing spooky action in tests.

### P1-T3 — Grounding endpoints in core-api
Implement `POST /projects/{id}/grounding`, `GET /grounding/{job_id}`, `GET /grounding/{job_id}/items` per `contracts/api-surface.md`. A background task (FastAPI `BackgroundTasks` is fine for now) polls adapter status and upserts items into `collected_items`; sets `grounding_jobs.status` and `projects.status` per data-model invariant 1.
**Done when:** integration test (`tests/integration/test_grounding.py`) with mocked adapter: create project → launch grounding → items appear in DB → status flips to `grounded`. Paste output.

**Files:**
- `services/core-api/app/routers/grounding.py` — the three endpoints:
  - `POST /projects/{id}/grounding` — validates body (pydantic schema `GroundingLaunchRequest`), inserts `grounding_jobs` row (id = uuid generated here and reused as adapter `job_id` — data-model.md fixes this), sets `projects.status='grounding'`, calls adapter `start_collect` with the contract §1 payload, registers the poller in `BackgroundTasks`, returns `{job_id}`.
  - `GET /grounding/{job_id}` — returns the `grounding_jobs` row's `status` + `progress` (fresh from the last poll; do not proxy-live on every GET).
  - `GET /grounding/{job_id}/items?source=&offset=&limit=` — reads `collected_items` from the DB (not the adapter), enforces `limit ≤ 200` (api-surface behavioral req 5).
- `services/core-api/app/services/grounding.py`:
  - `async def run_grounding(job_id: uuid.UUID) -> None` — the background loop: poll `get_status` until `done`/`failed` (with a generous max duration, e.g. 45 min); page `get_items` (limit 100) until `offset >= total`; upsert each page; update `grounding_jobs.progress`; on `done` set job + `projects.status='grounded'`; on adapter `failed` set job `failed` and `projects.status='failed'` (invariant 1 transitions only).
  - `def upsert_items(session, job_id, items: list[dict]) -> int` — `INSERT ... ON CONFLICT (item_id) DO NOTHING` (or UPDATE of mutable metrics); returns inserted count.
- `services/core-api/app/routers/projects.py` + `workspaces.py` — mt-01 steps 2–3 exercise `POST /workspaces/{wid}/projects` and `POST /projects/{id}/seed`; both exist in `contracts/api-surface.md` so they are contract-legal to add here if not already present. Seed upload: accept multipart ≤10 MB (PDF/MD/TXT), store the file in MinIO bucket `crowdlens-artifacts`, insert `seed_documents` version 1, return `{seed_id, version, preview_text}` (first ~500 chars of extracted text). PDF text extraction: use a pinned, documented library and name it in the PR description; if none is agreed, handle MD/TXT now and stop-and-report on PDF.
- `services/core-api/app/deps.py` — dev-mode identity: with `AUTH_DISABLED=true`, every request resolves to a synthetic owner + default dev workspace (api-surface RBAC note); a `get_dev_workspace()` dependency creates the row on first use.

**Tests:** `tests/integration/test_grounding.py` (adapter mocked via respx or dependency-override; compose postgres real)
- `test_full_grounding_flow` — the done-criteria test: create project → POST grounding → poller runs against mocked adapter → rows in `collected_items` → project `grounded`.
- `test_launch_sets_status_grounding_immediately` — before any poll completes.
- `test_items_upsert_deduplicates` — adapter returns the same item on two pages → one row.
- `test_items_endpoint_limit_capped_at_200` — `limit=500` → 400/422 VALIDATION or clamped (pick one, document).
- `test_items_endpoint_filters_by_source`.
- `test_adapter_failure_flips_status_failed` — mocked `status:"failed"` → job `failed`, project `failed`, no fake success (R10).
- `test_unknown_job_id_404` — error envelope `NOT_FOUND`.
- `test_grounding_launch_unknown_project_404`.

**Edge cases:**
- Duplicate items across pages (adapter paging overlap) — `item_id` PK + ON CONFLICT handles it; don't dedupe in Python.
- Poller crash mid-run leaves project stuck in `grounding` — catch-all in `run_grounding` flips to `failed` with the exception logged verbatim.
- Job `done` with zero items — still a valid `done`; project goes `grounded` and the *analyze* step will deal with thin data (do not fail here).
- Poller interval: don't hammer — 10–15s is plenty for a 30-min job.

**Failure modes (cheap LLM):**
- Polling synchronously inside the POST request — request hangs for 30 minutes; the 202-and-poll pattern exists for a reason.
- Flipping `projects.status` to `grounded` before items are committed — status becomes a lie (R10); order: commit items → commit job status → commit project status.
- Proxying `GET /grounding/{job_id}` live to the adapter on every console poll — the DB row is the cache; the console polls every 5s.
- Writing status transitions outside invariant 1 (e.g. `grounded → grounding` blocked silently instead of a clean `CONFLICT`).
- Inventing query params or response fields not in api-surface — R1.

### P1-T4 — Live collectors (real APIs)
Wire the fork's Reddit (PRAW) and YouTube collectors so `POST /collect` actually pulls live data per the contract payload (`keywords`, `time_window_days`, `max_items_per_source`). If the fork's native interface differs, the adapter translates — fork code stays untouched.
**Done when:** human runs the curl in `mt-01` step 4 and real rows (real URLs, real timestamps, no usernames) land in `collected_items`.

**Files:**
- `services/adapters/adapters/bettafish.py` — only if the fork's request/response fields differ from contract §1: add a translation shim (`def _to_fork_collect(payload: dict) -> dict`, `def _from_fork_item(raw: dict) -> dict`) inside the adapter. No new endpoints, no fork edits.
- `forks/bettafish/Dockerfile.adapter` — only if extra env passthrough is needed for the Reddit/YouTube keys.

**Tests:** no new automated tests — live-API behavior is verified by mt-01 steps 4–7 (real URLs, real timestamps, PII absence). A recorded-fixture regression test is optional; never substitute it for the live run.

**Edge cases:**
- Reddit 429s during a big pull — adapter must surface `RATE_LIMITED` (retryable) rather than crash the job; the fork may handle backoff itself, in which case the adapter just passes status through.
- YouTube daily quota exhaustion — explicit stop condition; surface it, never silently mock.
- Deleted/removed Reddit posts with empty text — drop them at the adapter (a `text NOT NULL` column can't hold them, and an empty item has no grounding value).
- `time_window_days` filtering: if the fork doesn't filter, filter by `published_at` in the adapter translation layer.
- Duplicate items across keywords (same post matches two keywords) — `item_id` (sha256 of platform+native_id) makes these converge to one row; verify the fork/adapter computes it that way.

**Failure modes (cheap LLM):**
- Mocking or fixture-faking the live pull to make the done-criteria "pass" — the stop condition explicitly forbids this; quota problems are reported, not hidden.
- Letting fork-native fields (usernames, avatars) ride through because "the fork returned them" — `_strip_pii` applies to live payloads exactly as to fixtures (R5).
- Patching the fork's collectors when the interface mismatches — translation belongs in the adapter (R2); a mismatch too big to translate is the stop condition.
- Treating `max_items_per_source` as a global cap or ignoring it — it's per source, per the contract name.

### P1-T5 — Analysis + citation verification
Implement `POST /grounding/{job_id}/analyze` and `GET /reports/baseline/{report_id}` per contract. **The citation verifier is core-api code, not fork code**: `app/services/citation_check.py` walks every `themes[].claim`, asserts ≥1 `citation_item_ids` entry, and asserts each cited id exists in `collected_items` for that job. Sets `baseline_reports.verified`. Unverified reports are never served (422).
**Done when:** test with a tampered fixture (claim citing a nonexistent item) → `verified=false`, endpoint 422; real analysis of phase-1-collected data → `verified=true`.

**Files:**
- `services/core-api/app/routers/grounding.py` — add `POST /grounding/{job_id}/analyze`: calls adapter `start_analyze`, inserts `baseline_reports` row (id = adapter's `report_id`, `verified=false`), background-polls `get_analysis`, on `done` stores `summary` and runs the verifier.
- `services/core-api/app/routers/reports.py` — `GET /reports/baseline/{report_id}`: 404 if unknown, **422 `INSUFFICIENT_GROUNDING` if `verified=false`**, else the report row.
- `services/core-api/app/services/citation_check.py`:
  - `def verify_report(session, job_id: uuid.UUID, summary: dict) -> bool` — for every `themes[]`: `claim` present AND `len(citation_item_ids) >= 1` AND every cited id ∈ `collected_items` **for that job_id** (one `SELECT ... WHERE job_id = ... AND item_id = ANY(...)` query, not N queries). Any violation → False. Sets `baseline_reports.verified` and commits.
- `services/core-api/app/routers/projects.py` (or a small `costs.py`) — mt-01 step 12 calls `GET /api/v1/projects/{id}/costs`; it exists in api-surface so implement the read-only version here: `SELECT component, SUM(cost_usd), SUM(tokens_in), SUM(tokens_out) FROM cost_ledger WHERE project_id=... GROUP BY component` → `{total, by_component, cap, remaining}`. (Analysis cost logging itself: record the adapter's reported token usage into `cost_ledger` with component `analysis` when the report completes.)
- `services/core-api/app/services/llm.py` — LiteLLM helper used from here on: `async def chat(model_alias: str, messages: list[dict]) -> ChatResult` posting to `{LITELLM_BASE_URL}/v1/chat/completions` with the master key (per-project virtual keys arrive in phase 3 — P3-T4), returning content + `usage`; `def record_cost(session, project_id, component, model_alias, usage) -> None` writing the `cost_ledger` row. Cost in USD: read it from LiteLLM's response/spend metadata per the LiteLLM docs; if unavailable, store tokens and `cost_usd` from LiteLLM's model pricing map — never invent a price (R4).

**Tests:** `tests/integration/test_analysis.py`
- `test_analyze_happy_path_verified_true` — mocked adapter returns a well-formed summary whose citations all exist → `verified=true`, endpoint serves it.
- `test_tampered_citation_sets_verified_false` — the done-criteria test: one claim cites a nonexistent item → `verified=false`, GET → 422 `INSUFFICIENT_GROUNDING`.
- `test_claim_with_empty_citation_list_fails`.
- `test_citation_from_other_job_fails` — id exists but under a different `job_id` → unverified (cross-job bleed check).
- `test_theme_without_claim_fails`.
- `test_report_not_found_404`.
- `test_unverified_report_never_served_even_with_flag` — no `?force=true` backdoor exists.
- `test_costs_endpoint_groups_by_component` — two ledger rows → correct `by_component` and `total`.

**Edge cases:**
- `themes: []` (empty analysis) — verifier passes vacuously but the report is useless; serve it, and let mt-01 step 9's human eyeball judge quality (don't invent a failure mode the contract doesn't define).
- Duplicate ids within one claim's `citation_item_ids` — harmless; existence check is set-based.
- Adapter analysis status `failed` — job stays queryable, report endpoint 422 (there is no verified report), error surfaced verbatim.
- Very large `summary` jsonb — store as-is; never truncate to fit.

**Failure modes (cheap LLM):**
- Checking only that `citation_item_ids` is non-empty, not that ids exist — that is exactly the tamper case mt-01 step 10 tests.
- Verifying against all of `collected_items` without the `job_id` filter — ids from older projects would falsely validate.
- "Repairing" unverified reports by dropping uncited claims and serving the rest — hides the grounding failure (Principle 1); unverified means 422, full stop.
- Trusting a `verified` flag coming from the adapter — verification is core-api's job, recomputed from the DB, every time.
- Returning 200 with `verified: false` in the body "so the UI can warn" — the contract says 422; the UI learns nothing from a body it never gets.

### P1-T6 — Grounding Console (frontend v0)
Plain Tailwind page at `/projects/:id/grounding`: launch form (keywords, sources, window), live progress per source (poll `GET /grounding/{job_id}` every 5s), table of 20 sample items (source, date, text excerpt, link), and the baseline report rendered with each claim followed by its cited links. No auth yet (`AUTH_DISABLED=true`).
**Done when:** `npm run build` clean; human completes mt-01 step 7 in the browser.

**Files:**
- `frontend/src/api/client.ts` — tiny fetch wrapper: `API_BASE = import.meta.env.VITE_API_BASE_URL ?? "http://localhost:8000/api/v1"`, JSON handling, throws the api-surface error envelope on non-2xx. Add `VITE_API_BASE_URL` to `.env.example`.
- `frontend/src/pages/GroundingConsole.tsx` — the page: launch form (keywords as comma-separated input, sources checkboxes reddit/youtube, time window number), progress section, items table, report section.
- `frontend/src/components/ProgressPerSource.tsx` — renders `progress.{source}.collected/target` bars; polls `GET /grounding/{job_id}` every 5s via `setInterval` inside `useEffect` **with cleanup**, stops polling when `status` is `done`/`failed`.
- `frontend/src/components/ItemsTable.tsx` — fetches `GET /grounding/{job_id}/items?limit=20`; columns source / date / excerpt (≤200 chars, truncated client-side) / link (`target="_blank" rel="noreferrer"`).
- `frontend/src/components/BaselineReport.tsx` — sentiment summary, themes list; each claim followed by its cited items resolved from the items endpoint (citation → clickable URL).
- Routing: the page lives at `/projects/:id/grounding`, which needs a client-side router. `react-router-dom` is **not** in the P0-T1 allowlist — add it (pinned) as a declared dependency of this task per R8, or stop and ask; do not hand-roll routing with `window.location` hacks.

**Tests:** no frontend test infra yet — `npm run build` clean + the mt-01 browser steps are the gate.

**Edge cases:**
- Component unmounts mid-poll (user navigates away) — interval must clear or it leaks and spams the API.
- `job_id` in URL but unknown → API 404 → render the error envelope message, not a blank page.
- Report not ready yet (analysis running) — render a "running" state; do not 404-crash the page on 422.
- Items with `published_at: null` — render "unknown date", never a crashed `toLocaleString` on null.
- Long excerpts with URLs/markup — render as plain text (`{item.text}`), never `dangerouslySetInnerHTML`.

**Failure modes (cheap LLM):**
- Hardcoding `localhost:8000` — use the env-driven base URL or the human's mt-01 run breaks the moment ports differ.
- Polling with `setInterval` but no cleanup, or polling forever after `done`.
- Resolving citations by array index instead of `item_id` lookup — items arrive paged; indices lie.
- Adding auth headers/UI "preemptively" — phase 3's job; `AUTH_DISABLED=true` means no token logic at all now.
- Reaching for a component library beyond the shadcn/ui set already scaffolded — plain Tailwind is the brief.

## Stop conditions
- Fork's collector interface can't produce the contract payload without fork changes → stop, report the gap precisely (which field, which endpoint).
- YouTube quota exhausted during dev → stop, tell the human; do not silently mock.

## Manual test
`../manual-tests/mt-01-collectors.md` — all green before phase 2.
