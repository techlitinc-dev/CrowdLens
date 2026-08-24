# Phase 2 — First Loop (the crown jewel)

**Goal:** one full decision cycle works end-to-end: seed doc → grounding → persona panel → counterfactual simulation ensemble → one-page verdict, in a minimal UI. **Gate:** `manual-tests/mt-02-first-loop.md` all green.

## Prerequisites
- Phase 1 green. Human adds MiroShark fork to `forks/miroshark` (`git clone <fork-url>`) and its `.env` values (Neo4j credentials matching compose, `LITELLM_BASE_URL`).

## Tasks

### P2-T1 — MiroShark service in compose
Same pattern as P1-T1: service `miroshark`, port `5002`, depends_on neo4j + litellm. Only new file allowed in fork: `Dockerfile.adapter`. MiroShark must be configured to route its LLM calls through LiteLLM (`swarm-model` alias) — if it only supports direct provider keys, stop and report (do not patch fork code yet).
**Done when:** healthy container; its API responds on `localhost:5002`.

**Files:**
- `infra/docker-compose.yml` — append service `miroshark`: `build: context: ../forks/miroshark`, `ports: ["5002:<fork-port>"]`, `networks: [crowdlens-net]`, `depends_on: neo4j: service_healthy, litellm: service_started`, env passthrough: `INTERNAL_SERVICE_TOKEN`, Neo4j URI/credentials **matching the compose neo4j service** (`bolt://neo4j:7687`, same password), and whatever base-URL var the fork reads for its LLM endpoint.
- `forks/miroshark/Dockerfile.adapter` — only if the fork ships no Dockerfile; same rules as P1-T1 (base image + start command per the fork's own docs).

**Tests:** none automated — mt-02 step 1 is the test.

**Edge cases:**
- The LLM base URL the fork sees must be `http://litellm:4000` (compose network name), **not** `http://localhost:4000` — localhost inside a container is the container.
- Neo4j credentials drift between compose and the fork's env = connection refused on first run; keep one source of truth in `.env`.
- The fork may expect a LiteLLM key — pass `LITELLM_MASTER_KEY` for now; per-project virtual keys are phase 3.

**Failure modes (cheap LLM):**
- Same classes as P1-T1: guessing healthcheck paths, patching fork code, hardcoding creds, publishing extra ports.
- Specific to this task: silently leaving the fork pointed at DeepSeek directly when it can't be configured for LiteLLM — that violates R3 and is the named stop condition, not a judgment call.

### P2-T2 — Adapter module: MiroShark
`services/adapters/adapters/miroshark.py` implementing `contracts/adapter-contract.md` §2 exactly: `create_simulation`, `get_status`, `get_results`, `ask`. Validate every outgoing handoff payload against the contract constraints (agent_count ≤150, rounds ≤30, ≤3 variants, proportions sum to 1.0±0.01) — raise `VALIDATION` before any HTTP call.
Unit tests: constraint rejection cases, response parsing, error envelope.
**Done when:** `pytest services/adapters` green.

**Files:**
- `services/adapters/adapters/miroshark.py`:
  - `class HandoffValidationError(Exception)` — carries a list of violated constraints; core-api maps it to the `VALIDATION` envelope.
  - `def validate_handoff(payload: dict) -> None` — pure function, no I/O: `config.agent_count ≤ 150` · `config.rounds ≤ 30` · `len(variants) ≤ 3` · `abs(sum(a.proportion for a in persona_panel.archetypes) - 1.0) ≤ 0.01` · required keys present (`simulation_id`, `reality_seed_md`, `persona_panel`, `variants`, `config`). Raises before any network activity.
  - `class MiroSharkAdapter` — same constructor/retry pattern as `BettaFishAdapter` (reuse the envelope parsing; a small shared `_envelope.py` helper inside the adapters package is fine — no new deps):
    - `async def create_simulation(self, payload: dict) -> dict` — calls `validate_handoff` first, then POST `/simulations`; returns `{simulation_id, run_ids}`.
    - `async def get_status(self, simulation_id: str) -> dict` — GET `/simulations/{id}/status`.
    - `async def get_results(self, simulation_id: str) -> dict` — GET `/simulations/{id}/results`.
    - `async def ask(self, simulation_id: str, run_id: str, agent_id: str, question: str) -> dict` — POST `/simulations/{id}/ask`.

**Tests:** `services/adapters/tests/test_miroshark.py` (respx-mocked)
- `test_validate_rejects_agent_count_151` / `test_validate_accepts_150`.
- `test_validate_rejects_rounds_31` / `test_validate_accepts_30`.
- `test_validate_rejects_four_variants`.
- `test_validate_rejects_proportions_sum_1_05`.
- `test_validate_accepts_proportions_sum_0_995` — the ±0.01 tolerance boundary.
- `test_validate_rejects_missing_config_key`.
- `test_create_simulation_invalid_payload_makes_no_http_call` — assert the respx router recorded **zero** requests (this is the "raise before any HTTP call" requirement, proven).
- `test_create_simulation_parses_run_ids`.
- `test_get_results_parses_runs_shape` — final_sentiment / stance_by_archetype / top_objections / narratives survive intact.
- `test_ask_response_has_simulated_true`.
- `test_error_envelope_mapped_to_adapter_error` — code + message verbatim.

**Edge cases:**
- Proportion rounding: `[0.333, 0.333, 0.333]` sums to 0.999 — must pass (within tolerance). Exact-equality comparison is a bug.
- Empty `archetypes: []` — sum is 0.0 → correctly rejected.
- Float noise: compute the sum with `math.fsum` or compare with the tolerance, never `== 1.0`.
- `ask` for a `run_id` that isn't in the status response — pass through; the fork's `NOT_FOUND` envelope is the answer, not client-side guessing.

**Failure modes (cheap LLM):**
- Validating after the POST (or logging-and-continuing) — the whole point is that an invalid handoff never leaves core-api.
- Clamping invalid values instead of rejecting (e.g. silently setting `agent_count=150`) — silent mutation hides caller bugs; raise.
- Implementing the tolerance as `round(sum) == 1`.
- Copy-pasting the BettaFish retry block and accidentally retrying non-retryable `INVALID_INPUT`.
- Adding fields to the handoff "MiroShark might want" — contract §2's schema is closed (R1).

### P2-T3 — Handoff Transformer (`app/services/handoff.py`)
**This is the highest-value file in the project. Work slowly. Human review required before merge.**
Input: verified `baseline_reports.summary` + sampled `collected_items` + seed document text + variants. Output: the exact handoff payload (contract §2), including:
- `reality_seed_md` — markdown with: facts section (claims + citation item_ids), sentiment summary, ≤20 verbatim quotes (each with its item_id), key entities.
- `persona_panel` — 4–8 archetypes; proportions derived from theme/segment frequency in the data (never invented — if data is too thin to segment, return 2 archetypes + flag `low_confidence_panel: true`).
- LLM drafting via `report-model` alias through LiteLLM; the project's virtual key; log tokens to `cost_ledger` (component `analysis`).
**Done when:** tests with fixture grounding data produce a payload that passes the P2-T2 validator; a human eyeballs one real payload and approves the seed doc quality.

**Files:**
- `services/core-api/app/services/handoff.py`:
  - `async def build_handoff(session, project_id: uuid.UUID, variants: list[dict]) -> dict` — orchestrates: load latest verified baseline report + its `job_id` → `sample_items` → `derive_panel` → `draft_reality_seed` → assemble payload → `validate_handoff` (imported from the adapters package) → return. Refuses (raises → 422 `INSUFFICIENT_GROUNDING`) if no verified report exists — invariant 2 enforced at the source.
  - `def sample_items(session, job_id: uuid.UUID, limit: int = 200) -> list[dict]` — deterministic sample: order by `metrics->>'score'` desc NULLS LAST, take top N, so re-runs of the same job produce the same panel (human review needs reproducibility).
  - `def derive_panel(summary: dict, items: list[dict]) -> tuple[dict, bool]` — proportions from theme/segment frequency: count items per theme (`item_ids` membership), normalize; returns `(panel, low_confidence)`. Thin-data rule: if fewer than ~30 items or fewer than 2 themes with ≥5 items each → 2 archetypes + `low_confidence_panel: true`. Archetype `language_samples` are `text` values copied verbatim from sampled items, each annotated with its `item_id`; `llm_model: "swarm-model"` on every archetype.
  - `async def draft_reality_seed(summary, items, seed_text: str | None, low_confidence: bool) -> str` — LLM draft via `app/services/llm.py` `chat("report-model", ...)`; prompt supplies the claims, sentiment, entities, and the verbatim quote pool and instructs the model to use only those. Returns the markdown.
  - `def verify_seed(seed_md: str, items: list[dict]) -> None` — post-check: every quote attributed to an `item_id` in the seed must be a substring of that item's actual `text`; every `item_id` mentioned must exist in the sample set. Any violation → raise (caller redrafts once, then fails honestly — R10).
- `services/core-api/app/services/llm.py` — from P1-T5; here the `chat` call must use the per-project virtual key when `projects.litellm_key` is set, master key otherwise (keys arrive in P3-T4; until then every project is on the master key — note this in the code comment so phase 3 doesn't miss the swap point). Every call is followed by `record_cost(..., component="analysis")`.

**Tests:** `services/core-api/tests/test_handoff.py` (unit, fixtures under `tests/fixtures/grounding/`)
- `test_payload_passes_validator` — done-criteria: fixture grounding data → `validate_handoff` does not raise.
- `test_proportions_track_theme_frequency` — skewed fixture (theme A has 3× the items of theme B) → archetype proportions skew the same direction; never uniform-by-default.
- `test_thin_data_two_archetypes_low_confidence` — 10-item fixture → 2 archetypes, flag set.
- `test_language_samples_are_verbatim` — every sample string appears in the fixture items' texts.
- `test_seed_quotes_verbatim_postcheck` — a doctored seed whose quote paraphrases the source → `verify_seed` raises.
- `test_seed_rejects_invented_item_ids`.
- `test_seed_quote_cap_20` — fixture with 50 candidate quotes → seed cites ≤20.
- `test_cost_logged_with_component_analysis` — mocked LiteLLM response with `usage` → one `cost_ledger` row.
- `test_no_verified_report_raises_insufficient_grounding`.
- `test_seed_document_optional` — project without a seed doc still builds (seed_text=None → question text only).

**Edge cases:**
- A claim's cited item fell outside the top-N sample — pull cited items into the sample **additionally** (union), so the seed never references an item the payload can't support.
- Quotes containing markdown (`*`, backticks, `|`) — the seed is markdown; escape or blockquote them, don't strip characters (verbatim means verbatim).
- Items longer than the model context — truncate the item list, never truncate a quote mid-string.
- All items in one theme → 2-archetype low-confidence path, not a forced 4-archetype panel.
- `low_confidence_panel: true` is a payload-level flag; `validate_handoff` must tolerate it (it's additive, not a contract violation).

**Failure modes (cheap LLM):**
- The LLM paraphrasing quotes to "flow better" — this is precisely why `verify_seed` exists; never ship the drafter without the post-check, and never make the post-check warn-and-continue.
- Inventing proportions (0.25/0.25/0.25/0.25) because uniform "looks fair" — proportions come from counts or the panel is fake (R4, Principle 1).
- Letting the model introduce facts not in the summary/seed — the prompt gives it source material only; a seed "fact" with no citation item_id is a stop condition (phase-2 stop #3).
- Swallowing a failed first draft and silently returning partial markdown — redraft once, then raise with the verbatim error.
- Forgetting the `cost_ledger` row — mt-01 step 12 / mt-02 step 11 audit this.

### P2-T4 — Persona panel endpoints
`POST /persona-panel/generate`, `GET`, `PUT` per api-surface. `PUT` stores a **new version row** (never mutate), renormalizes proportions, reuses the P2-T2 validator.
**Done when:** integration test: generate → edit proportion → new version row exists, old row unchanged.

**Files:**
- `services/core-api/app/routers/personas.py`:
  - `POST /projects/{id}/persona-panel/generate` — runs P2-T3's `build_handoff`-adjacent panel path (reuse `sample_items` + `derive_panel`; the LLM may draft archetype names/descriptions from the same material), stores `persona_panels` version = `MAX(version)+1` for the project, `edited_by=null` (auto-generated), returns `{panel_id, version}`.
  - `GET /projects/{id}/persona-panel` — latest version by `(project_id, version desc)`.
  - `PUT /persona-panels/{panel_id}` — body `{panel}`: load the addressed row (for project + shape), `renormalize` proportions, `validate_handoff`-style panel validation (reuse the P2-T2 constraint checks on the panel portion), insert a **new** row with next version and `edited_by` = current user (dev user while `AUTH_DISABLED`); the old row is untouched.
- `services/core-api/app/services/panel.py`:
  - `def renormalize(panel: dict) -> dict` — divide each proportion by the sum; then fix residual rounding drift by adjusting the largest archetype so the sum lands within 1.0±0.01 deterministically.
  - `def next_version(session, project_id) -> int` — `SELECT COALESCE(MAX(version),0)+1 ...`.

**Tests:** `tests/integration/test_persona_panel.py`
- `test_generate_creates_version_1` — done-criteria half 1.
- `test_put_creates_version_2_old_row_unchanged` — done-criteria half 2: read back version 1, byte-compare `panel` jsonb.
- `test_put_renormalizes_proportions` — edit sets one proportion to 0.9 → stored row sums to 1.0±0.01.
- `test_put_rejects_unfixable_panel` — proportions `[0,0,0]` → 400 `VALIDATION`, no new row.
- `test_get_returns_latest_version`.
- `test_put_unknown_panel_404`.

**Edge cases:**
- Concurrent PUTs racing for `MAX(version)+1` — the `UNIQUE (project_id, version)` constraint is the guard; catch `IntegrityError`, retry once, else surface `CONFLICT`. Don't `SELECT ... FOR UPDATE`-free double-insert.
- Renormalization of all-zero proportions is impossible → reject, never divide-by-zero into NaN (jsonb would store `NaN`, which is invalid JSON).
- Human edits a proportion to negative → validator rejects.
- Editing an old version's row by id: PUT addresses a specific panel row but always appends the new version to that row's project — document the behavior, don't silently fork panels.

**Failure modes (cheap LLM):**
- Updating the existing row in place — the done-criteria explicitly tests immutability; version history is what makes panel edits auditable.
- Renormalizing by naive division and storing a sum like 0.9999999997 without the drift fix — fine for the validator (tolerance) but the mt-02 step 4 eyeball expects clean numbers; deterministic fix-up keeps diffs readable.
- Reimplementing validation inline instead of reusing P2-T2's — two validators will drift apart by phase 4.
- Forgetting `edited_by=null` on auto-generated rows vs the dev user on PUT — the data model uses that column to tell them apart.

### P2-T5 — Simulation orchestrator + adaptive ensemble
`app/services/ensemble.py` implements adapter contract §3 verbatim: 3 runs per variant → agreement check (modal direction + ≥2/3 top-objection overlap) → extend to max 7 → `verdict` or `no_consensus`. Orchestration via FastAPI background task polling adapter status (Temporal comes later — do not add it now). Budget pre-flight per data-model invariant 3; mid-run, poll LiteLLM spend for the project key and halt at cap (`halted_budget`).
**Done when:** integration tests with mocked adapter prove: (a) 3 agreeing runs → verdict with `agreement_score`; (b) 3 disagreeing + 2 more agreeing → verdict at 5 runs; (c) 7 disagreeing → `no_consensus`, no verdict row with `outcome='verdict'`; (d) spend cap hit mid-run → `halted_budget`.

**Files:**
- `services/core-api/app/routers/simulations.py`:
  - `POST /projects/{id}/simulations` — pre-flight in order: (1) invariant 2: verified baseline report exists for latest grounding job, else 422 `INSUFFICIENT_GROUNDING`; (2) invariant 3: budget pre-flight, else 402 `BUDGET_EXCEEDED` + project `halted_budget`; (3) build handoff via P2-T3, `validate_handoff`, insert `simulations` row (id generated here, reused as adapter `simulation_id`), set project `simulating`, call adapter `create_simulation`, register ensemble background task, return `{simulation_id}`.
  - `GET /simulations/{sid}` — adapter status shape + `cost_to_date` from `cost_ledger` (api-surface).
  - `GET /simulations/{sid}/results` — 409/422 until the ensemble is complete (pick one code from the api-surface enum and document it); then per-variant aggregated results.
  - `POST /simulations/{sid}/ask` — pass-through to adapter `ask`; response always carries `simulated: true` (api-surface).
- `services/core-api/app/services/ensemble.py`:
  - `def run_direction(run_results: dict) -> str` — `'support' | 'oppose' | 'neutral'` from `final_sentiment` / `stance_by_archetype` majority; define the mapping in one place and unit-test it.
  - `def objection_overlap(a: list[str], b: list[str]) -> float` — overlap fraction of top-3 objection themes (set intersection over 3).
  - `def compute_agreement(runs: list[dict]) -> float` — contract §3 rule 2: fraction of runs sharing the modal direction AND ≥2/3 top-objection overlap with the modal set.
  - `async def run_ensemble(simulation_id: uuid.UUID) -> None` — the loop: initial 3 runs/variant → poll statuses → when all terminal: `compute_agreement` → ≥0.8 → finalize `verdict`; <0.8 and total runs <7 → `create_simulation` for extension runs (ensemble_index increments); 7 without convergence → finalize `no_consensus` with a divergence summary. Failed runs: replace a `failed` run once (same ensemble_index slot freed), a second failure → project `failed` with the adapter error verbatim.
  - `def finalize(simulation_id, outcome, agreement_score) -> None` — writes the `verdicts` row skeleton (`outcome`, `agreement_score`; `ranked_variants` etc. filled by P2-T6 for `verdict` outcome only) and flips project status to `verdict_ready`/`no_consensus` (invariant 1).
- `services/core-api/app/services/budget.py`:
  - `def estimate_simulation_cost(config: dict) -> Decimal` — derived from agent_count × rounds × per-token pricing of `swarm-model` as reported by LiteLLM's model info; if pricing is unavailable, use a conservative constant documented in a comment — and mark the estimate as such (R4: no invented precision).
  - `def preflight(session, project_id, estimate) -> None` — invariant 3 exactly: `SUM(cost_ledger.cost_usd) + estimate > spend_cap_usd` → raise (→ 402 + `halted_budget`).
  - `async def check_spend_mid_run(session, project_id) -> bool` — sums `cost_ledger` for the project (component `simulation`) against the cap; called from the ensemble poll loop; over cap → stop requesting new runs, set `halted_budget`, record why. (Note: per-project LiteLLM keys don't exist until P3-T4, so in phase 2 "poll LiteLLM spend for the project key" is implemented as cost_ledger sums; leave a `TODO(phase-3)` comment pointing at the key-based spend API swap.)

**Tests:** `tests/integration/test_ensemble.py` (adapter mocked; postgres real) — the done-criteria set:
- `test_three_agreeing_runs_verdict` — (a): three runs per variant with same direction + overlapping objections → `verdicts` row, `outcome='verdict'`, `agreement_score ≥ 0.8`, project `verdict_ready`.
- `test_extension_to_five_runs_converges` — (b): 3 disagreeing then 2 agreeing → verdict, and assert exactly 5 run slots were requested.
- `test_seven_runs_no_consensus` — (c): `outcome='no_consensus'`, and `SELECT ... WHERE outcome='verdict'` returns nothing; project status `no_consensus`.
- `test_budget_cap_mid_run_halts` — (d): cap lowered between polls → project `halted_budget`, no new `create_simulation` calls after the trip.
- `test_preflight_over_cap_402` — invariant 3: 402 envelope and `halted_budget` before any adapter call.
- `test_no_verified_baseline_422` — invariant 2 / api-surface behavioral req 1 (mirrors mt-02 step 6).
- `test_objection_overlap_two_of_three_counts` — the ≥2/3 rule unit-level.
- `test_failed_run_replaced_once_then_project_failed`.
- `test_results_409_until_ensemble_complete`.

**Edge cases:**
- Modal direction tie (support == oppose counts) — define deterministically (e.g. tie → the ensemble cannot have agreement ≥0.8 on direction; extend runs) and cover it in a test comment.
- Early-stopped runs (fewer rounds) still count as full runs — `early_stopped: true` is data, not a failure.
- Extension runs must reuse the identical handoff payload except `ensemble_index` — changing the seed mid-ensemble makes agreement meaningless.
- Poll loop lifetime: cap total wall-clock (e.g. 3h) and fail honestly rather than poll forever.
- The 402 pre-flight flips status to `halted_budget` **and** returns the error — both, per api-surface behavioral req 2.

**Failure modes (cheap LLM):**
- Computing agreement on sentiment magnitudes instead of the contract's modal-direction + objection-overlap rule — §3 is verbatim law, not inspiration.
- Off-by-one on the run cap: launching an 8th run, or stopping at 6 — max is exactly 7.
- Writing a `verdicts` row with `outcome='verdict'` for a diverged ensemble "so the UI has something" — never force a verdict (§3 rule 3; mt-02 step 9 checks the DB directly).
- Asking MiroShark about convergence — ensemble logic lives in core-api only (api-surface behavioral req 3).
- Adding Temporal "because it's in the stack" — explicitly deferred; background tasks now.
- Mid-run budget check reading LiteLLM spend under the master key with no project scoping (keys arrive in phase 3) — cost_ledger sums are the honest phase-2 source.

### P2-T6 — Verdict engine v0 (`app/services/verdict.py`)
Second crown-jewel file. Input: converged ensemble results + baseline report. Output per `verdicts` table: `ranked_variants` (by support direction — NO invented magnitudes beyond what runs report), `objections` (each quote tagged `origin: 'real'` with item_id, or `origin: 'sim'` with run ref), `risk_flags`, `confidence` from agreement score (data-model invariant 4), `report_markdown` — one page: recommendation, ranked options, top objections with quotes, risks, confidence + convergence. Drafted by `report-model`; **every claim in the markdown must reference a verdict field** (post-check: any paragraph without a reference is removed).
**Done when:** test fixtures → markdown contains zero unreferenced paragraphs; `GET /projects/{id}/verdict` serves it.

**Files:**
- `services/core-api/app/services/verdict.py`:
  - `async def build_verdict(session, simulation_id: uuid.UUID) -> None` — called by `ensemble.finalize` on `verdict` outcome: loads runs' results + the verified baseline report → fills `ranked_variants`, `objections`, `risk_flags`, `confidence` → drafts markdown → post-check → UPDATE the skeleton verdicts row. For `no_consensus`, only the divergence summary markdown is drafted — never ranked data.
  - `def rank_variants(runs: list[dict]) -> list[dict]` — `[{variant_id, label, direction, support}]`; `direction` from `run_direction` majority across the variant's runs; `support` only the aggregated values runs actually report (e.g. mean of `stance.support` fractions present in results) — if runs report no magnitude, the field is `null` with the direction still shown (R4).
  - `def collect_objections(runs, baseline) -> list[dict]` — merge run `top_objections` (origin `sim`, ref = `run_id:agent:round:post` quote_ref from contract §2) with baseline themes (origin `real`, ref = `item_id`); dedupe by theme label; cap at top 5.
  - `def confidence_from_agreement(score: Decimal) -> str` — invariant 4 exactly: `>=0.9 → 'high'`, `>=0.8 → 'medium'`, below → unreachable (no verdict allowed), raise if called with it.
  - `async def draft_markdown(fields: dict) -> str` — `chat("report-model", ...)` with the verdict fields as the only source material; log `cost_ledger` component `verdict`.
  - `def strip_unreferenced_paragraphs(md: str, fields: dict) -> str` — split on blank lines; a paragraph survives only if it references at least one verdict field (variant label, objection theme, quote text fragment, confidence word, or agreement score as written in `fields`). Removed paragraphs are logged, not silently dropped (R10-adjacent: the human reviewing logs sees what was cut).
- `services/core-api/app/routers/projects.py` — `GET /projects/{id}/verdict`: latest `verdicts` row for the project (any outcome — api-surface says the payload includes the `NO_CONSENSUS` state); 404 if none yet.

**Tests:** `tests/integration/test_verdict.py`
- `test_markdown_zero_unreferenced_paragraphs` — done-criteria: fixture verdict with a planted hallucinated paragraph ("The market is worth $5B") → stripped; final md has none.
- `test_confidence_mapping` — 0.92→high, 0.83→medium, 0.79→raise.
- `test_ranked_variants_directions_only_from_runs` — fixture runs all "oppose" for variant B → B ranked last with direction oppose; assert no magnitude keys appear that runs didn't report.
- `test_objections_origin_tags` — real objections carry `origin:'real'` + item_id resolvable in `collected_items`; sim carry `origin:'sim'` + run ref matching a `sim_runs.id`.
- `test_no_consensus_markdown_has_no_ranked_data` — divergence summary only.
- `test_get_verdict_serves_latest` — two simulations → latest row served.
- `test_get_verdict_404_before_any_simulation`.
- `test_verdict_cost_logged` — component `verdict` row in `cost_ledger`.

**Edge cases:**
- Real-data objection and sim objection share a theme → merge into one entry with both quote kinds; never drop the real one (evidence-first).
- Variant tie in support → keep input order and say "tied" in the markdown; do not coin a tiebreaker metric.
- All paragraphs get stripped (bad draft) → redraft once; if still empty, store fields and a minimal templated markdown stating recommendation + confidence without decoration — a terse honest page beats an eloquent fake one.
- `agreement_score` stored as `numeric(4,3)` — round at the boundary, don't let float repr leak into the DB.
- A newer simulation exists while an old verdict is being served — `GET` always serves the latest; no caching.

**Failure modes (cheap LLM):**
- The drafter adding an "executive summary" with invented magnitudes ("adoption will rise 15%") — the explicit R4 trap; the post-check exists because prompts alone won't stop it.
- Implementing the post-check as "warn but keep" — removal is the specified behavior.
- Tagging sim quotes as real (or omitting `origin`) because the theme matched a real one — mt-02 step 8/12 eyeballs exactly this.
- Hardcoding `confidence='medium'` or deriving it from run count instead of `agreement_score` — invariant 4 is the only mapping.
- Presenting ranked variants for `no_consensus` — mt-02 step 9 checks the UI shows no verdict-ranked data in that state.

### P2-T7 — Minimal UI: wizard + verdict page
- `/projects/new`: 3-step wizard (question+seed upload → sources → budget cap). Calls existing endpoints.
- `/projects/:id/verdict`: renders the verdict — confidence meter, ranked variant bars, objection list expandable to quotes (real quotes show platform+date+link; sim quotes show a "SIMULATED" badge). Dark theme, Tailwind, nothing fancy.
**Done when:** `npm run build` clean; human walks mt-02 step 8 in the browser.

**Files:**
- `frontend/src/pages/NewProjectWizard.tsx` — 3 steps, local state machine (`step: 1|2|3`), no form library: (1) name + question + seed file upload → `POST /workspaces/{wid}/projects` then `POST /projects/{id}/seed`; (2) sources + keywords + window → stores locally; (3) budget cap → PATCH-less (cap is set at project create; if the user edits it at step 3, re-POST is impossible — collect cap at step 1's request body instead; simplest: ask cap on step 1, steps 2–3 review/launch grounding via `POST /projects/{id}/grounding`). Resolve the dev workspace via `GET /workspaces` first entry while `AUTH_DISABLED`.
- `frontend/src/pages/VerdictPage.tsx` — polls `GET /simulations/{sid}` (via `GET /projects/{id}` for status) while `simulating`, renders verdict when `verdict_ready`, renders the `no_consensus` state honestly when that's the outcome.
- `frontend/src/components/ConfidenceMeter.tsx` — high/medium/low + `agreement_score` number.
- `frontend/src/components/VariantBars.tsx` — bar per variant; bar length from `support` when non-null, otherwise direction label only — never a fabricated length.
- `frontend/src/components/ObjectionList.tsx` — expandable rows; real quote → platform (source) + date + outbound link; sim quote → `SIMULATED` badge, no link.
- `frontend/src/api/client.ts` — add the new endpoint functions; same envelope-throwing wrapper.

**Tests:** `npm run build` clean + mt-02 step 8. No component-test infra this phase.

**Edge cases:**
- Verdict page opened while `simulating` — live progress view, poll with cleanup (same leak rules as P1-T6).
- Outcome `no_consensus` — show divergence summary and NO ranked bars (mt-02 step 9 checks this).
- `support: null` on a variant — render direction only.
- Quote with missing date/platform — render "unknown", keep the link if present.
- Seed file >10 MB — server rejects; wizard surfaces the envelope message verbatim, not a generic "upload failed".

**Failure modes (cheap LLM):**
- Showing ranked variants / a winner in the `no_consensus` state — the single most-checked honesty property of this page.
- Forgetting the SIMULATED badge or styling it subtly — it must be unmistakable (mt-02 step 8).
- Inventing bar magnitudes from array indices or confidence words — lengths come from `support` or nothing.
- Re-uploading the seed on every wizard back-navigation — steps keep client state; the seed POST happens once.
- Adding charts/auth/onboarding polish — "nothing fancy" is the brief; dark theme + Tailwind only.

## Stop conditions
- MiroShark can't accept the persona panel shape → stop; the fix is an adapter translation layer, not a fork patch — report first.
- Ensemble cost exceeds $3 in the P2-T5 manual test → stop; tune rounds/agents down per §11.1 levers and report.
- Any LLM-drafted content (seed doc, verdict) shows invented citations → stop; this is a P0 bug, escalate to human.

## Manual test
`../manual-tests/mt-02-first-loop.md` — the full cycle, run by a human, all green before phase 3.
