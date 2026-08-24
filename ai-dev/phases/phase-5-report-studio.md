# Phase 5 — Report Studio

**Goal:** living, evidence-linked reports: block editor, publish validation, share links, white-label exports, ask-the-report. **Gate:** `manual-tests/mt-05-report-studio.md` green.

## Prerequisites
- Phase 4 green. `contracts/report-blocks.md` is law for this phase. Add dep: `pgvector` extension in postgres (migration), WeasyPrint + python-pptx (worker), MinIO bucket `crowdlens-exports`.

## Tasks

### P5-T1 — Report model + CRUD
Migration creating `reports`, `report_shares`, `share_views`, `report_embeddings`, `workspace_branding` **exactly as defined in `contracts/data-model.md` → Later-phase tables** (never redefine schema in this file). Endpoints per report-blocks §API (POST/GET/PUT). PUT = new version row.
**Done when:** integration tests: create → edit → version increments, old version immutable.

**Files:**
- `services/core-api/alembic/versions/0005_reports.py` — creates `reports`, `report_shares`, `share_views`, `report_embeddings`, `workspace_branding` verbatim from data-model.md → Later-phase tables (note: `workspace_branding` sits under the "Phase 8" comment there but is created now — phase 5 uses it for white-label exports), plus `CREATE EXTENSION IF NOT EXISTS vector;`.
- `app/models.py` — SQLAlchemy models matching 1:1.
- `app/routers/reports.py` — POST/GET/PUT per report-blocks §API.
- `app/services/report_store.py` — version logic in one place.

**Key functions:**
- `async def create_report(project_id: UUID, vertical: str, user_id: UUID) -> Report` — template's default block set per `vertical`.
- `async def put_report(report_id: UUID, title: str, blocks: list[dict], user_id: UUID) -> Report` — always inserts a new version row (`version = max+1` per `UNIQUE (project_id, version, language)`); the old row is never mutated.
- `def validate_block_set(blocks: list[dict]) -> None` — rejects any `type` outside the closed set in report-blocks.md → `VALIDATION`.

**Tests** (`tests/integration/test_reports_crud.py`): `test_create_from_vertical_template`, `test_put_creates_new_version_row`, `test_old_version_row_immutable`, `test_unknown_block_type_rejected`, `test_concurrent_put_conflict`, `test_language_variants_share_version_sequence`.

**Edge cases:** concurrent PUTs → one wins, loser gets `CONFLICT`; block `id`s preserved across versions so evidence rails and ask-citations stay stable; PUT on an `archived` report → `CONFLICT`, not a silent resurrection; version numbers never reuse after archive.

**Cheap-LLM failure modes:** none at CRUD level — blocks are stored as given; quality gates are P5-T2's job.

### P5-T2 — Publish validation pipeline
`app/services/report_validate.py`: walks every block, enforces the evidence rules from the contract (every quote/number → resolvable evidence_ref). Then the Langfuse claim-evidence eval (`report-model`): sample each claim, verify the cited evidence actually supports it; score < 0.9 → publish blocked with per-block reasons. This is a hard gate, not a warning.
**Done when:** tests: tampered evidence_ref → 422 with block ids; eval mock failing → blocked; valid report → published.

**Files:**
- `app/services/report_validate.py` — stage 1, deterministic walk (no LLM).
- `app/services/claim_eval.py` — stage 2, the Langfuse claim-evidence eval.
- `app/routers/reports.py` — `POST /reports/{rid}/publish` wiring the two stages.

**Key functions:**
- `def validate_blocks(blocks: list[dict], db) -> list[BlockFailure]` — every quantitative/quoted statement has ≥1 `evidence_ref`; every ref resolves **within this project** (item exists / sim post exists in the project's runs). Failure list carries block ids.
- `async def claim_evidence_eval(report: Report) -> EvalResult` — samples each claim, asks `report-model` whether the cited evidence supports it; mean score < 0.9 → blocked with per-block reasons. Every attempt traced in Langfuse.
- `async def publish_report(report_id: UUID) -> Report` — stage 1 → stage 2 → status `published`; any failure → 422, status stays `draft`.

**Tests** (`tests/integration/test_report_publish.py`): `test_quote_without_evidence_ref_422_names_block`, `test_tampered_item_id_ref_422`, `test_ref_to_other_projects_item_rejected`, `test_eval_below_threshold_blocks_with_reasons`, `test_eval_timeout_fails_closed`, `test_valid_report_publishes`, `test_markdown_block_without_claims_needs_no_refs`.

**Edge cases:** ref to an item from a different job/project → rejected (resolve check is project-scoped); near-verbatim quotes with whitespace drift → eval prompt must tolerate them, not false-block (see stop condition on flakiness); eval model timeout/5xx → publish **fails closed**, never auto-passes; `cost_ledger` block is computed at render time and exempt from ref rules per contract (read-only).

**Cheap-LLM failure modes:** the eval must run on `report-model`, never `swarm-model` — a cheap grader rubber-stamps unsupported claims (grounded-or-nothing); JSON-mode drift in eval output → one parse retry, then fail closed; long reports → batch claims per block so nothing is silently truncated out of the sample.

### P5-T3 — Share links
`POST /reports/{rid}/share` + public `GET /share/{token}`: view-only renderer route (read-only block renderer, no editor chrome), expiry enforced server-side, watermark (viewer IP + timestamp) in footer, view events logged for `share_viewed` analytics (phase 7 wires Novu; now: store).
**Done when:** mt-05 steps 4–5 (incl. expired link → 410).

**Files:**
- `app/routers/share.py` — `POST /reports/{rid}/share` (member-authed) and `GET /share/{token}` (public, **no auth dependency** — keep it off the JWT middleware).
- `app/services/shares.py` — token lifecycle + view logging.
- `frontend/src/routes/share/SharedReportPage.tsx` — view-only renderer: read-only block components, no editor chrome, watermark footer.

**Key functions:**
- `async def create_share(report_id: UUID, expires_at: datetime, user_id: UUID) -> {share_url, expires_at}` — token = 32 random bytes, shown once; only `sha256(token)` in `token_hash`; expiry >30d rejected at creation (data-model invariant 5).
- `async def get_shared_report(token: str, viewer_ip: str) -> SharedView` — hash lookup; missing → 404, expired → **410** (mt-05 step 5); logs a `share_views` row.
- Since each report version is its own row, `report_shares.report_id` already pins the shared version — later edits don't mutate what viewers see.

**Tests** (`tests/integration/test_share_links.py`): `test_token_never_stored_plaintext`, `test_expiry_over_30d_rejected`, `test_expired_link_returns_410`, `test_share_renders_only_published_version`, `test_view_logged_with_ip`, `test_share_of_draft_report_refused`, `test_share_view_writes_audit_event`.

**Edge cases:** watermark = viewer IP + timestamp rendered from `watermark_seed` at view time (never stored per-viewer — PII rule); clock skew on the 1-minute dev-override expiry (mt-05 step 5) — compare in UTC, reject `expires_at <= now()` at creation; `viewer_ip` behind the dev proxy → use `X-Forwarded-For` first hop or socket peer, document which; `inet` column rejects garbage → store NULL rather than 500.

**Cheap-LLM failure modes:** none — no LLM in this path.

### P5-T4 — Exports (PDF/PPTX)
Export worker (FastAPI background task is fine until phase 8): PDF via WeasyPrint from a print stylesheet; PPTX via python-pptx (title slide, one slide per block type, speaker notes carry evidence refs). White-label: workspace branding fields (logo storage key, accent color) applied; SIMULATED badges and cost ledger preserved per contract. Output → MinIO, signed URL (15 min).
**Done when:** mt-05 step 6 — open both files, verify branding, badges, cost ledger.

### P5-T5 — Ask-the-report (RAG)
pgvector: embed project artifacts (baseline report, verdict markdown, block texts, item excerpts) with the LiteLLM embedding alias; `POST /reports/{rid}/ask` retrieves top-k (scoped to project_id — hard filter, not similarity), answers via `report-model`, and **must cite block ids + evidence refs**; no citation → answer refused ("insufficient grounded data in this report").
**Done when:** tests: cross-project leakage attempt returns nothing; uncited answers refused; mt-05 step 7.

### P5-T6 — Branch what-ifs
From any `sim_comparison` block: "rerun with…" → dialog collecting a new `seed_patch_md` → creates a new simulation on the same project (existing endpoint, same caps/ensemble rules) → on completion, a new block version links old vs new.
**Done when:** mt-05 step 8.

### P5-T7 — Report Studio UI
`/projects/:id/reports/:rid`: block canvas (add/remove/reorder blocks, drag-and-drop), per-block settings panel, evidence rail on hover, publish button (shows validation failures inline), share/export/ask buttons. Verdict page `/projects/:id/verdict` becomes a rendered report view of the auto-generated verdict report.
**Done when:** mt-05 full pass in browser.

## Stop conditions
- The claim-evidence eval is flaky (false blocks on valid reports) → tune the eval prompt once; if still flaky, stop and report — never weaken the gate to pass.
- PPTX layout work is ballooning → implement title + bullet slides only, note the limitation, move on.

## Manual test
`../manual-tests/mt-05-report-studio.md`
