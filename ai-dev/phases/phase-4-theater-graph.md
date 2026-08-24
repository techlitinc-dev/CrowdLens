# Phase 4 — Simulation Theater & Knowledge Graph

**Goal:** the wow layer — live simulation viewing, agent interview UI, persona editor, and the graph explorer. **Gate:** `manual-tests/mt-04-theater-graph.md` green.

## Prerequisites
- Phases 0–3 green. `contracts/websocket-protocol.md` and `contracts/frontend-spec.md` read by the agent before any UI task.

## Tasks

### P4-T1 — WebSocket gateway in core-api
Implement `contracts/websocket-protocol.md` exactly: `POST /simulations/{sid}/ws-ticket` (single-use, 60s TTL), the WS endpoint, `hello` snapshot, `resume`, event proxying from MiroShark, ensemble_update from core-api state, backpressure gap handling. `simulated: true` on every post/ask event — fail closed.
**Done when:** integration test with mocked MiroShark events: connect → snapshot → 3 rounds → reconnect with `resume` → no duplicates, gap flag works; a post event missing `simulated:true` is never emitted.

### P4-T2 — Simulation Theater UI
`/projects/:id/simulation` per frontend-spec: branch tabs (one per variant), live post feed (virtualized, framer-motion entry, SIMULATED badges), round scrubber (drag to any past round → REST refetch), per-archetype sentiment lanes (ECharts, fed by `sentiment_tick`). Reconnecting badge on disconnect.
**Done when:** `npm run build` clean; mt-04 step 3 (live run in browser) passes.

### P4-T3 — Agent interview UI
Ask box on any agent's post → `ask` WS message → answer rendered inline with purple SIMULATED treatment + run/round context link.
**Done when:** mt-04 step 4.

### P4-T4 — Persona Panel Editor UI
`/projects/:id/personas`: archetype cards (name, stance chip, description, language samples linking to real items), proportion sliders that auto-renormalize, edit → `PUT` (new version — existing endpoint), "regenerate" → generate endpoint. Show `low_confidence_panel` warning when set by the handoff transformer.
**Done when:** mt-04 step 5 (edit flow) passes.

### P4-T5 — Graph read API
core-api endpoints proxying Neo4j (read-only): `GET /projects/{id}/graph/entities?q=&from=&to=` (search + time window), `GET /graph/entities/{eid}` (detail + linked evidence: real items and sim posts), `GET /projects/{id}/graph/snapshot?at=` (before/after toggle support). All scoped to the project's simulation ids — never expose another tenant's graph.
**Done when:** integration tests incl. cross-tenant isolation test.

### P4-T6 — Knowledge Graph Explorer UI
`/projects/:id/graph`: react-force-graph canvas, search box, click node → focus mode + evidence drawer, time scrubber (uses snapshot endpoint), before/after toggle. Performance: viewport culling, smooth at 5K nodes — test with a generated fixture graph (`tests/fixtures/graph-5k.json`).
**Done when:** mt-04 step 7 incl. the 5K-node smoothness check.

### P4-T7 — Home + wizard polish
Upgrade phase-2 v0 screens to frontend-spec: project cards (status, cost, confidence), full 5-step wizard with defaults. Dark theme tokens applied app-wide.
**Done when:** mt-04 steps 1–2.

## Stop conditions
- MiroShark event stream doesn't match the WS contract → translate in core-api; never leak fork shapes to the frontend. If untranslatable, stop and report.
- 5K-node performance target unreachable with react-force-graph → stop, report measured numbers, propose fallback (deck.gl) — don't swap libraries silently.

## Manual test
`../manual-tests/mt-04-theater-graph.md`
