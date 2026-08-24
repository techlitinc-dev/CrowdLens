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

### P1-T2 — Adapter module: BettaFish
`services/adapters/adapters/bettafish.py`: async httpx client implementing every endpoint in `contracts/adapter-contract.md` §1 with the exact payload shapes. Includes `_strip_pii(item)` applied to every collected item: removes `author`, `username`, `avatar`, profile URLs before anything is stored. Retries: 3× exponential backoff on `retryable: true` errors only.
Unit tests with `respx`-mocked HTTP: happy path, PII stripping proof, error envelope mapping.
**Done when:** `pytest services/adapters` green; a test asserts a fixture item containing `author: "someuser"` is stored without it.

### P1-T3 — Grounding endpoints in core-api
Implement `POST /projects/{id}/grounding`, `GET /grounding/{job_id}`, `GET /grounding/{job_id}/items` per `contracts/api-surface.md`. A background task (FastAPI `BackgroundTasks` is fine for now) polls adapter status and upserts items into `collected_items`; sets `grounding_jobs.status` and `projects.status` per data-model invariant 1.
**Done when:** integration test (`tests/integration/test_grounding.py`) with mocked adapter: create project → launch grounding → items appear in DB → status flips to `grounded`. Paste output.

### P1-T4 — Live collectors (real APIs)
Wire the fork's Reddit (PRAW) and YouTube collectors so `POST /collect` actually pulls live data per the contract payload (`keywords`, `time_window_days`, `max_items_per_source`). If the fork's native interface differs, the adapter translates — fork code stays untouched.
**Done when:** human runs the curl in `mt-01` step 4 and real rows (real URLs, real timestamps, no usernames) land in `collected_items`.

### P1-T5 — Analysis + citation verification
Implement `POST /grounding/{job_id}/analyze` and `GET /reports/baseline/{report_id}` per contract. **The citation verifier is core-api code, not fork code**: `app/services/citation_check.py` walks every `themes[].claim`, asserts ≥1 `citation_item_ids` entry, and asserts each cited id exists in `collected_items` for that job. Sets `baseline_reports.verified`. Unverified reports are never served (422).
**Done when:** test with a tampered fixture (claim citing a nonexistent item) → `verified=false`, endpoint 422; real analysis of phase-1-collected data → `verified=true`.

### P1-T6 — Grounding Console (frontend v0)
Plain Tailwind page at `/projects/:id/grounding`: launch form (keywords, sources, window), live progress per source (poll `GET /grounding/{job_id}` every 5s), table of 20 sample items (source, date, text excerpt, link), and the baseline report rendered with each claim followed by its cited links. No auth yet (`AUTH_DISABLED=true`).
**Done when:** `npm run build` clean; human completes mt-01 step 7 in the browser.

## Stop conditions
- Fork's collector interface can't produce the contract payload without fork changes → stop, report the gap precisely (which field, which endpoint).
- YouTube quota exhausted during dev → stop, tell the human; do not silently mock.

## Manual test
`../manual-tests/mt-01-collectors.md` — all green before phase 2.
