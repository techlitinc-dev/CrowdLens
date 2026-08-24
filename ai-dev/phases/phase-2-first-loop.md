# Phase 2 — First Loop (the crown jewel)

**Goal:** one full decision cycle works end-to-end: seed doc → grounding → persona panel → counterfactual simulation ensemble → one-page verdict, in a minimal UI. **Gate:** `manual-tests/mt-02-first-loop.md` all green.

## Prerequisites
- Phase 1 green. Human adds MiroShark fork to `forks/miroshark` (`git clone <fork-url>`) and its `.env` values (Neo4j credentials matching compose, `LITELLM_BASE_URL`).

## Tasks

### P2-T1 — MiroShark service in compose
Same pattern as P1-T1: service `miroshark`, port `5002`, depends_on neo4j + litellm. Only new file allowed in fork: `Dockerfile.adapter`. MiroShark must be configured to route its LLM calls through LiteLLM (`swarm-model` alias) — if it only supports direct provider keys, stop and report (do not patch fork code yet).
**Done when:** healthy container; its API responds on `localhost:5002`.

### P2-T2 — Adapter module: MiroShark
`services/adapters/adapters/miroshark.py` implementing `contracts/adapter-contract.md` §2 exactly: `create_simulation`, `get_status`, `get_results`, `ask`. Validate every outgoing handoff payload against the contract constraints (agent_count ≤150, rounds ≤30, ≤3 variants, proportions sum to 1.0±0.01) — raise `VALIDATION` before any HTTP call.
Unit tests: constraint rejection cases, response parsing, error envelope.
**Done when:** `pytest services/adapters` green.

### P2-T3 — Handoff Transformer (`app/services/handoff.py`)
**This is the highest-value file in the project. Work slowly. Human review required before merge.**
Input: verified `baseline_reports.summary` + sampled `collected_items` + seed document text + variants. Output: the exact handoff payload (contract §2), including:
- `reality_seed_md` — markdown with: facts section (claims + citation item_ids), sentiment summary, ≤20 verbatim quotes (each with its item_id), key entities.
- `persona_panel` — 4–8 archetypes; proportions derived from theme/segment frequency in the data (never invented — if data is too thin to segment, return 2 archetypes + flag `low_confidence_panel: true`).
- LLM drafting via `report-model` alias through LiteLLM; the project's virtual key; log tokens to `cost_ledger` (component `analysis`).
**Done when:** tests with fixture grounding data produce a payload that passes the P2-T2 validator; a human eyeballs one real payload and approves the seed doc quality.

### P2-T4 — Persona panel endpoints
`POST /persona-panel/generate`, `GET`, `PUT` per api-surface. `PUT` stores a **new version row** (never mutate), renormalizes proportions, reuses the P2-T2 validator.
**Done when:** integration test: generate → edit proportion → new version row exists, old row unchanged.

### P2-T5 — Simulation orchestrator + adaptive ensemble
`app/services/ensemble.py` implements adapter contract §3 verbatim: 3 runs per variant → agreement check (modal direction + ≥2/3 top-objection overlap) → extend to max 7 → `verdict` or `no_consensus`. Orchestration via FastAPI background task polling adapter status (Temporal comes later — do not add it now). Budget pre-flight per data-model invariant 3; mid-run, poll LiteLLM spend for the project key and halt at cap (`halted_budget`).
**Done when:** integration tests with mocked adapter prove: (a) 3 agreeing runs → verdict with `agreement_score`; (b) 3 disagreeing + 2 more agreeing → verdict at 5 runs; (c) 7 disagreeing → `no_consensus`, no verdict row with `outcome='verdict'`; (d) spend cap hit mid-run → `halted_budget`.

### P2-T6 — Verdict engine v0 (`app/services/verdict.py`)
Second crown-jewel file. Input: converged ensemble results + baseline report. Output per `verdicts` table: `ranked_variants` (by support direction — NO invented magnitudes beyond what runs report), `objections` (each quote tagged `origin: 'real'` with item_id, or `origin: 'sim'` with run ref), `risk_flags`, `confidence` from agreement score (data-model invariant 4), `report_markdown` — one page: recommendation, ranked options, top objections with quotes, risks, confidence + convergence. Drafted by `report-model`; **every claim in the markdown must reference a verdict field** (post-check: any paragraph without a reference is removed).
**Done when:** test fixtures → markdown contains zero unreferenced paragraphs; `GET /projects/{id}/verdict` serves it.

### P2-T7 — Minimal UI: wizard + verdict page
- `/projects/new`: 3-step wizard (question+seed upload → sources → budget cap). Calls existing endpoints.
- `/projects/:id/verdict`: renders the verdict — confidence meter, ranked variant bars, objection list expandable to quotes (real quotes show platform+date+link; sim quotes show a "SIMULATED" badge). Dark theme, Tailwind, nothing fancy.
**Done when:** `npm run build` clean; human walks mt-02 step 8 in the browser.

## Stop conditions
- MiroShark can't accept the persona panel shape → stop; the fix is an adapter translation layer, not a fork patch — report first.
- Ensemble cost exceeds $3 in the P2-T5 manual test → stop; tune rounds/agents down per §11.1 levers and report.
- Any LLM-drafted content (seed doc, verdict) shows invented citations → stop; this is a P0 bug, escalate to human.

## Manual test
`../manual-tests/mt-02-first-loop.md` — the full cycle, run by a human, all green before phase 3.
