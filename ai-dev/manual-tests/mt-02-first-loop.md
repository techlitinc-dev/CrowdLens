# MT-02 — First Loop (full decision cycle)

**Prereq:** MT-01 green with real collected data. This test runs a real simulation — budget ~60–90 min wall time.

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | `docker compose up -d miroshark` → `curl localhost:5002/` (or `/health`) | Service responds; Neo4j browser (localhost:7474) shows MiroShark graph activity later during runs | |
| 2 | Generate panel: `curl -X POST .../projects/<pid>/persona-panel/generate` | 4–8 archetypes; proportions sum to ~1.0; each archetype has real `language_samples` | |
| 3 | **Panel quality eyeball:** read the generated `reality_seed_md` + panel in psql (`SELECT panel FROM persona_panels ...`) | Quotes are verbatim from collected items (spot-check 3 against `collected_items.text`); no invented "facts"; archetypes plausibly match the data | |
| 4 | Edit panel via `PUT /api/v1/persona-panels/<panel_id>` (change one proportion) | New version row created; old version untouched; proportions renormalized to 1.0 | |
| 5 | Launch simulation: `curl -X POST .../projects/<pid>/simulations -d '{"variants":[{"label":"Price ₹29,999","seed_patch_md":"The price is ₹29,999."},{"label":"Price ₹39,999","seed_patch_md":"The price is ₹39,999."}]}'` | `simulation_id` returned; project status → `simulating` | |
| 6 | **Grounding gate (negative test):** create a second project with NO grounding, POST a simulation on it | `422 INSUFFICIENT_GROUNDING` — no simulation possible without verified grounding | |
| 7 | Poll `GET /api/v1/simulations/<sid>` every 2 min | 3 ensemble runs per variant start; `current_round` advances; runs terminate ≤18 rounds (early-stop allowed) | |
| 8 | Browser: `/projects/<pid>/verdict` while running, then when done | Live status visible; final verdict shows confidence meter, ranked variants, objections; every real quote has platform+date+link; every sim quote has a SIMULATED badge | |
| 9 | **Convergence check:** `SELECT outcome, agreement_score, confidence FROM verdicts WHERE project_id='<pid>'` | If `outcome='verdict'`: agreement ≥0.8, confidence matches score (≥0.9 high, ≥0.8 medium). If `no_consensus`: NO verdict-ranked data is presented in UI | |
| 10 | **Agent interview:** `curl -X POST .../simulations/<sid>/ask -d '{"run_id":"<rid>","agent_id":"<aid>","question":"Why did you oppose the price?"}'` | Answer returns with `"simulated": true` and the run reference | |
| 11 | **MONEY CHECK (the big one):** `GET /api/v1/projects/<pid>/costs` + LiteLLM spend dashboard | Simulation component ≤ **$3** (§11.1 target); all 4 components recorded; total ≤ $25 cap with huge margin. If sim cost >$3 → phase-2 stop condition: tune agents/rounds down | |
| 12 | **Verdict honesty eyeball:** read `report_markdown` top to bottom | No invented magnitudes (no "CTR will be 3.2%"); every claim traces to a run or a citation; uncertainty stated where agreement was thin | |
| 13 | **Budget halt (negative test):** set the project's `spend_cap_usd` to just above current spend in psql, launch another simulation with 1 variant | Either `402 BUDGET_EXCEEDED` upfront, or project flips to `halted_budget` mid-run when the cap trips | |

**Sign-off:** all PASS → the product's core loop works. Steps 9, 11, 12 are the trust-critical ones — a FAIL there blocks phase 3 unconditionally.
