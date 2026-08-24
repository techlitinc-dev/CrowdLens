# Phase 5 — Report Studio

**Goal:** living, evidence-linked reports: block editor, publish validation, share links, white-label exports, ask-the-report. **Gate:** `manual-tests/mt-05-report-studio.md` green.

## Prerequisites
- Phase 4 green. `contracts/report-blocks.md` is law for this phase. Add dep: `pgvector` extension in postgres (migration), WeasyPrint + python-pptx (worker), MinIO bucket `crowdlens-exports`.

## Tasks

### P5-T1 — Report model + CRUD
Migration creating `reports`, `report_shares`, `share_views`, `report_embeddings`, `workspace_branding` **exactly as defined in `contracts/data-model.md` → Later-phase tables** (never redefine schema in this file). Endpoints per report-blocks §API (POST/GET/PUT). PUT = new version row.
**Done when:** integration tests: create → edit → version increments, old version immutable.

### P5-T2 — Publish validation pipeline
`app/services/report_validate.py`: walks every block, enforces the evidence rules from the contract (every quote/number → resolvable evidence_ref). Then the Langfuse claim-evidence eval (`report-model`): sample each claim, verify the cited evidence actually supports it; score < 0.9 → publish blocked with per-block reasons. This is a hard gate, not a warning.
**Done when:** tests: tampered evidence_ref → 422 with block ids; eval mock failing → blocked; valid report → published.

### P5-T3 — Share links
`POST /reports/{rid}/share` + public `GET /share/{token}`: view-only renderer route (read-only block renderer, no editor chrome), expiry enforced server-side, watermark (viewer IP + timestamp) in footer, view events logged for `share_viewed` analytics (phase 7 wires Novu; now: store).
**Done when:** mt-05 steps 4–5 (incl. expired link → 410).

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
