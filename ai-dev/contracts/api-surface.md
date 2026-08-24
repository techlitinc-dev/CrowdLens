# Core API Surface — REST v1

**Version 0.1 · FastAPI · Base path `/api/v1` · All responses JSON · Auth: Logto JWT bearer (phase 3); phases 0–2 run with `AUTH_DISABLED=true` and a synthetic dev user.**

Error envelope (all endpoints):
```json
{ "error": { "code": "STRING_ENUM", "message": "human-readable" } }
```
Codes: `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION`, `INSUFFICIENT_GROUNDING`, `BUDGET_EXCEEDED`, `CONFLICT`, `INTERNAL`.

Phases 0–2 RBAC: when `AUTH_DISABLED=true`, every request acts as an owner on a default dev workspace. Phase 3 replaces this with real JWT + role checks — endpoint behavior must not change.

---

## Workspaces

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/workspaces` | `{name}` → workspace | creator becomes `owner` |
| GET | `/workspaces` | → `[workspaces]` | caller's memberships only |
| POST | `/workspaces/{id}/invite` | `{email, role}` → `{invite_id}` | owner only |
| GET | `/workspaces/{id}/members` | → `[{user, role}]` | member only |

## Projects

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/workspaces/{wid}/projects` | `{name, question, spend_cap_usd}` → project | analyst+ |
| GET | `/workspaces/{wid}/projects` | → `[{id, name, status, cost_to_date, confidence}]` | cards for home screen |
| GET | `/projects/{id}` | → project detail incl. status history | |
| POST | `/projects/{id}/seed` | multipart file (PDF/MD/TXT) → `{seed_id, version, preview_text}` | ≤10 MB; text extracted server-side |

## Grounding

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/projects/{id}/grounding` | `{keywords[], sources[], region, languages[], time_window_days}` → `{job_id}` | sets status `grounding`; calls adapter POST /collect |
| GET | `/grounding/{job_id}` | → status + progress (adapter shape) | polled by Grounding Console |
| GET | `/grounding/{job_id}/items?source=&offset=&limit=` | → paged items | sample quotes for console |
| POST | `/grounding/{job_id}/analyze` | → `{report_id}` | adapter POST /analyze |
| GET | `/reports/baseline/{report_id}` | → baseline report, **only if citation-verified** | else `422 INSUFFICIENT_GROUNDING` |

## Personas

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/projects/{id}/persona-panel/generate` | → `{panel_id, version}` | `report-model` via LiteLLM; input = verified baseline report + item samples |
| GET | `/projects/{id}/persona-panel` | → latest panel | |
| PUT | `/persona-panels/{panel_id}` | `{panel}` → new version | proportions renormalized; stored as new row |

## Simulation

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/projects/{id}/simulations` | `{variants: [{label, seed_patch_md}]}` → `{simulation_id}` | runs all pre-flight checks (data-model invariants 2–3); launches adaptive ensemble per adapter §3 |
| GET | `/simulations/{sid}` | → status (adapter shape + cost_to_date) | |
| GET | `/simulations/{sid}/results` | → per-variant aggregated results | only when ensemble complete |
| POST | `/simulations/{sid}/ask` | `{run_id, agent_id, question}` → answer | pass-through to adapter; `simulated: true` always in response |
| WS | `/ws/simulations/{sid}` | stream: `{type: "round", run_id, round, posts: [...]}` | live theater feed; proxy of MiroShark events |

## Verdicts & cost

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| GET | `/projects/{id}/verdict` | → latest verdict (incl. `NO_CONSENSUS` state) | one-page payload for verdict UI |
| GET | `/projects/{id}/costs` | → `{total, by_component: {...}, cap, remaining}` | from `cost_ledger` |

---

## Behavioral requirements (test these)

1. `POST /projects/{id}/simulations` without a verified baseline report → `422 INSUFFICIENT_GROUNDING`. (Principle 1, no exceptions.)
2. Same call when `cost_to_date + estimate > spend_cap_usd` → `402 BUDGET_EXCEEDED` and project status `halted_budget`.
3. Ensemble logic lives in core-api (adapter §3) — MiroShark knows nothing about convergence.
4. Every write endpoint records into `cost_ledger` when an LLM call happened (read spend from LiteLLM response headers / spend API).
5. Pagination: `offset`/`limit` everywhere, `limit` ≤ 200.

---

## Extended surface (defined in later contracts — index only)

| Endpoints | Defined in | Phase |
|---|---|---|
| `POST /simulations/{sid}/ws-ticket`, `WS /ws/simulations/{sid}` | websocket-protocol.md | 4 |
| `GET /projects/{id}/graph/entities`, `/graph/entities/{eid}`, `/graph/snapshot` | phase-4 task P4-T5 | 4 |
| `POST/GET/PUT /reports*`, `/share/{token}`, `/reports/{rid}/export`, `/reports/{rid}/ask` | report-blocks.md | 5 |
| `GET /workspaces/{id}/audit` | phase-3 task P3-T5 | 3 |
| `POST /workspaces/{id}/api-keys`, public read API + webhook management | phase-7 task P7-T6 | 7 |
| `GET /projects/{id}/narratives`, monitoring schedule CRUD | phase-7 tasks P7-T1/T3 | 7 |

If an endpoint is needed that appears nowhere in this index: STOP — it must be added to a contract first (R1).
