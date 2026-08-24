# Phase 4 — Simulation Theater & Knowledge Graph

**Goal:** the wow layer — live simulation viewing, agent interview UI, persona editor, and the graph explorer. **Gate:** `manual-tests/mt-04-theater-graph.md` green.

## Prerequisites
- Phases 0–3 green. `contracts/websocket-protocol.md` and `contracts/frontend-spec.md` read by the agent before any UI task.

## Tasks

### P4-T1 — WebSocket gateway in core-api
Implement `contracts/websocket-protocol.md` exactly: `POST /simulations/{sid}/ws-ticket` (single-use, 60s TTL), the WS endpoint, `hello` snapshot, `resume`, event proxying from MiroShark, ensemble_update from core-api state, backpressure gap handling. `simulated: true` on every post/ask event — fail closed.
**Done when:** integration test with mocked MiroShark events: connect → snapshot → 3 rounds → reconnect with `resume` → no duplicates, gap flag works; a post event missing `simulated:true` is never emitted.

**Files:**
- `app/routers/simulations.py` — add `POST /simulations/{sid}/ws-ticket` (JWT auth, workspace membership checked at issuance — not at WS connect).
- `app/ws/tickets.py` — ticket store (redis or in-process with TTL), single-use consume.
- `app/ws/gateway.py` — the WS endpoint itself.
- `app/ws/proxy.py` — MiroShark event translation into contract events; the only file allowed to know fork shapes.

**Key functions:**
- `async def issue_ticket(user_id: UUID, simulation_id: UUID) -> str` — random ≥128-bit token, TTL 60s, bound to (user, simulation).
- `async def simulation_ws(ws: WebSocket, simulation_id: UUID, ticket: str)` — consume ticket → send `hello` snapshot → proxy loop → handle `resume`/`ask`.
- `def translate_event(raw: dict) -> dict | None` — fork shape → contract event; `None` = untranslatable, logged + dropped, never forwarded raw.
- `def fail_closed(event: dict) -> bool` — any `round`/`ask_answer` payload where posts lack `simulated: true` is dropped before emit (websocket-protocol hard rule 1).

**Tests** (`tests/integration/test_ws_gateway.py`): `test_ticket_single_use_second_connect_rejected`, `test_ticket_expires_after_60s`, `test_hello_snapshot_contains_runs_and_last_50_posts`, `test_resume_replays_without_duplicates`, `test_gap_beyond_500_rounds_sends_fresh_hello`, `test_post_missing_simulated_dropped`, `test_backpressure_drops_round_events_never_terminal`, `test_gap_flag_sent_for_dropped_range`, `test_ticket_for_other_tenants_simulation_403`.

**Edge cases:** reconnect mid-round with out-of-order arrivals — server replays by `round` order, client buffers ±2 per contract; JWT never appears in the WS URL (ticket only); `ask` rate limit 10/min per user enforced server-side; MiroShark disconnects mid-run → keep client socket alive, send `error` event, resume when the fork stream returns.

**Cheap-LLM failure modes:** no LLM call in this layer, but swarm-model garbage upstream arrives as malformed fork events — `translate_event` must drop + log them, never crash the socket or leak raw fork payloads.

### P4-T2 — Simulation Theater UI
`/projects/:id/simulation` per frontend-spec: branch tabs (one per variant), live post feed (virtualized, framer-motion entry, SIMULATED badges), round scrubber (drag to any past round → REST refetch), per-archetype sentiment lanes (ECharts, fed by `sentiment_tick`). Reconnecting badge on disconnect.
**Done when:** `npm run build` clean; mt-04 step 3 (live run in browser) passes.

**Files:**
- `frontend/src/routes/simulation/TheaterPage.tsx` — route shell, branch tabs keyed by `variant_id`.
- `frontend/src/routes/simulation/PostFeed.tsx` — virtualized list, framer-motion `layout` entry, `PostCard` with SIMULATED badge in `--sim` purple.
- `frontend/src/routes/simulation/RoundScrubber.tsx` — drag → pause live tail → REST refetch of that round → release resumes live.
- `frontend/src/routes/simulation/SentimentLanes.tsx` — ECharts, one lane per archetype, fed by `sentiment_tick`.
- `frontend/src/hooks/useSimulationFeed.ts` — the only WS consumer (frontend-spec rule 6): ticket fetch, connect, resume, reconnect badge state, gap → REST refetch.

**Key functions:**
- `function useSimulationFeed(simulationId: string): { posts, rounds, sentiment, status, reconnecting, ask }` — owns the socket lifecycle; buffers out-of-order rounds ±2 per contract.

**Tests:** no frontend unit harness yet — gate is `npm run build` + mt-04 steps 3 and 6 (disconnect/reconnect without duplicates).

**Edge cases:** ticket TTL (60s) expiring during a long offline stretch → fetch a fresh ticket on reconnect, never reuse; scrubbing while `round` events still arrive → incoming events buffer, don't yank the view; 20 posts/s perf budget (frontend-spec) — virtualization is mandatory, not optional; `prefers-reduced-motion` disables entry animation; post missing `simulated:true` renders nothing (fail closed mirrors the server).

**Cheap-LLM failure modes:** none client-side — the UI renders whatever the gateway emits and never "fixes" agent text.

### P4-T3 — Agent interview UI
Ask box on any agent's post → `ask` WS message → answer rendered inline with purple SIMULATED treatment + run/round context link.
**Done when:** mt-04 step 4.

**Files:**
- `frontend/src/components/AskAgentBox.tsx` — ask input attached to any post; client-side rate-limit guard (10/min, mirrors server).
- `frontend/src/components/AskAnswerInline.tsx` — purple SIMULATED treatment, run/round context link into the scrubber.
- `frontend/src/hooks/useSimulationFeed.ts` — exposes `ask({run_id, agent_id, question})` over the existing socket; answers cached keyed by `post_id` so navigation doesn't lose them.

**Tests:** mt-04 step 4 in browser; plus gateway-level `test_ask_rate_limit_429_error_event` in `test_ws_gateway.py` (P4-T1 file).

**Edge cases:** ask on a finished/failed run → server `error` event rendered inline as an honest failure, never a spinner that runs forever (frontend-spec rule 4); answer arriving after route change → cached, shown on return; question empty/whitespace → send button disabled.

**Cheap-LLM failure modes:** swarm-model may answer out-of-character or ungrounded — the UI renders answers **verbatim** with SIMULATED treatment (no cleanup, no paraphrase); if the adapter returns an error envelope, show the failure, not a fabricated answer.

### P4-T4 — Persona Panel Editor UI
`/projects/:id/personas`: archetype cards (name, stance chip, description, language samples linking to real items), proportion sliders that auto-renormalize, edit → `PUT` (new version — existing endpoint), "regenerate" → generate endpoint. Show `low_confidence_panel` warning when set by the handoff transformer.
**Done when:** mt-04 step 5 (edit flow) passes.

**Files:**
- `frontend/src/routes/personas/PersonaEditorPage.tsx` — route shell, version selector (old versions read-only).
- `frontend/src/routes/personas/ArchetypeCard.tsx` — name, stance chip, description, language samples linking to real `collected_items`.
- `frontend/src/routes/personas/ProportionSliders.tsx` — auto-renormalizing slider group.
- `frontend/src/components/LowConfidencePanelBanner.tsx` — shown when `low_confidence_panel: true`.

**Key functions:**
- `function renormalize(proportions: number[]): number[]` — client mirror of the server rule (sum 1.0 ± 0.01 per adapter contract); server stays authoritative, the PUT validator reuses P2-T2's checks.

**Tests:** mt-04 step 5 in browser; integration coverage lives in phase 2 (`PUT` creates a new version, old row unchanged).

**Edge cases:** float drift after many slider drags — renormalize on every change, not on save; editing while a simulation is running — the simulation's `panel_id` pins its version, so editing is safe; UI warns but doesn't block; "regenerate" confirms before calling the generate endpoint (it spends LLM money against the cap); samples whose `item_id` no longer resolves → warning chip, never a fake link.

**Cheap-LLM failure modes:** regenerate (report-model via LiteLLM) can return archetypes with invented quotes or proportions that don't sum — server-side validator rejects invalid payloads and the UI shows the rejection reason; never auto-patch a malformed panel into place.

### P4-T5 — Graph read API
core-api endpoints proxying Neo4j (read-only): `GET /projects/{id}/graph/entities?q=&from=&to=` (search + time window), `GET /graph/entities/{eid}` (detail + linked evidence: real items and sim posts), `GET /projects/{id}/graph/snapshot?at=` (before/after toggle support). All scoped to the project's simulation ids — never expose another tenant's graph.
**Done when:** integration tests incl. cross-tenant isolation test.

**Files:**
- `app/routers/graph.py` — the three endpoints; all behind `require_role('owner','analyst','viewer')` (read, any member) + project-membership check.
- `app/services/neo4j_read.py` — all Cypher lives here; parameterized queries only, never string-interpolated user input.

**Key functions:**
- `async def search_entities(project_id: UUID, q: str, from_ts, to_ts) -> list[Entity]` — text search + time window, scoped to the project's simulation ids.
- `async def entity_detail(project_id: UUID, entity_id: str) -> EntityDetail` — node + linked evidence: real `collected_items` and sim posts (`run_id`, `post_id`, `round`).
- `async def graph_snapshot(project_id: UUID, at: datetime) -> GraphSnapshot` — nodes/edges visible at `at`, powering the before/after toggle.

**Tests** (`tests/integration/test_graph_api.py`, Neo4j testcontainer or compose instance seeded with fixture graph): `test_search_scoped_to_project`, `test_empty_q_validation_error`, `test_entity_detail_links_real_and_sim_evidence`, `test_snapshot_at_changes_graph`, `test_cross_tenant_entity_returns_404_not_data`, `test_neo4j_down_503_names_component`.

**Edge cases:** cross-tenant entity id → **404, not 403** (don't confirm the entity exists); empty `q` → `VALIDATION`, never a full-graph dump; snapshot at a timestamp before the simulation started → empty graph, not an error; huge snapshots → cap nodes returned + tell the client it was truncated.

**Cheap-LLM failure modes:** none — read-only Cypher; any garbage in the graph was written by MiroShark upstream and is displayed as-is with evidence links.

### P4-T6 — Knowledge Graph Explorer UI
`/projects/:id/graph`: react-force-graph canvas, search box, click node → focus mode + evidence drawer, time scrubber (uses snapshot endpoint), before/after toggle. Performance: viewport culling, smooth at 5K nodes — test with a generated fixture graph (`tests/fixtures/graph-5k.json`).
**Done when:** mt-04 step 7 incl. the 5K-node smoothness check.

**Files:**
- `frontend/src/routes/graph/GraphExplorerPage.tsx` — route shell + react-force-graph canvas.
- `frontend/src/routes/graph/TimeScrubber.tsx` — drives `GET .../graph/snapshot?at=`.
- `frontend/src/components/EvidenceDrawer.tsx` — shared evidence drawer (real items vs sim posts, per frontend-spec rule 1).
- `scripts/gen_graph_fixture.py` — generates `tests/fixtures/graph-5k.json` (deterministic seed, 5K nodes / ~15K edges).

**Key functions:**
- `function cullToViewport(nodes, viewport) → nodes` — viewport culling before render; this is the 5K-node budget mechanism (frontend-spec performance budget).

**Tests:** `npm run build` clean; mt-04 steps 7–8 (fixture smoothness + cross-tenant probe at step 9).

**Edge cases:** node with zero evidence links → drawer shows "no evidence" state, never a blank panel; search matching many nodes → result list, first match focused; toggling before/after while a snapshot fetch is in flight → last toggle wins (cancel stale TanStack Query requests); truncated snapshot response → visible "showing N of M nodes" note, no silent omission.

**Cheap-LLM failure modes:** none — no LLM in this path.

### P4-T7 — Home + wizard polish
Upgrade phase-2 v0 screens to frontend-spec: project cards (status, cost, confidence), full 5-step wizard with defaults. Dark theme tokens applied app-wide.
**Done when:** mt-04 steps 1–2.

**Files:**
- `frontend/src/routes/home/HomePage.tsx` — project cards from `GET /workspaces/{wid}/projects` (status, `cost_to_date`, confidence chip).
- `frontend/src/routes/wizard/` — one component per wizard step (question → audience → sources → variants → budget cap), ≤5 steps.
- `frontend/src/theme/tokens.css` — verify tokens applied app-wide; remove any leftover hard-coded colors from the v0 screens.

**Tests:** `npm run build` clean; mt-04 steps 1–2 in browser.

**Edge cases:** confidence chip hidden (not "—") when no verdict exists; budget-cap step required, default prefilled from the workspace cap's remaining headroom; wizard abandoned mid-way → local state survives a refresh; `cost_to_date` renders from `cost_ledger` sums only — no derived-but-unsourced numbers (mt-04 step 10 honesty sweep covers these screens).

**Cheap-LLM failure modes:** none — no LLM in this path.

## Stop conditions
- MiroShark event stream doesn't match the WS contract → translate in core-api; never leak fork shapes to the frontend. If untranslatable, stop and report.
- 5K-node performance target unreachable with react-force-graph → stop, report measured numbers, propose fallback (deck.gl) — don't swap libraries silently.

## Manual test
`../manual-tests/mt-04-theater-graph.md`
