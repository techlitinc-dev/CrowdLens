# Core API Surface — REST v1

**Version 0.1 · FastAPI · Base path `/api/v1` · All responses JSON · Auth: Logto JWT bearer (phase 3); phases 0–2 run with `AUTH_DISABLED=true` and a synthetic dev user.**

Error envelope (all endpoints):
```json
{ "error": { "code": "STRING_ENUM", "message": "human-readable" } }
```
Codes: `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION`, `INSUFFICIENT_GROUNDING`, `BUDGET_EXCEEDED`, `CONFLICT`, `INTERNAL`.

HTTP status ↔ code mapping (fixed):

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `VALIDATION` | malformed body/params, business-rule input errors (FastAPI's default 422 request-parse errors are normalized to this so 422 keeps a single meaning) |
| 401 | `UNAUTHORIZED` | missing / invalid / expired JWT (phase 3+) |
| 402 | `BUDGET_EXCEEDED` | spend cap would be exceeded |
| 403 | `FORBIDDEN` | authenticated, but role or membership insufficient |
| 404 | `NOT_FOUND` | resource missing **or not visible to caller** (no existence leaks across workspaces) |
| 409 | `CONFLICT` | state conflict (job already running, duplicate membership, ensemble incomplete) |
| 413 | `VALIDATION` | payload too large (seed upload) |
| 422 | `INSUFFICIENT_GROUNDING` | grounding / citation gate failed (Principle 1) |
| 500 | `INTERNAL` | unexpected server error |

Phases 0–2 RBAC: when `AUTH_DISABLED=true`, every request acts as an owner on a default dev workspace. Phase 3 replaces this with real JWT + role checks — endpoint behavior must not change.

**Auth annotation convention below:** roles are Logto org roles `owner` / `analyst` / `viewer` (phase-3 matrix in `phases/phase-3-auth-billing.md` P3-T2). `analyst+` = owner or analyst. `member+` = any role with membership in the owning workspace. All endpoints use the user JWT bearer; the phase-7 service-token read API (`read:projects`, `read:verdicts` scopes) is a separate surface indexed below. curl examples show the phase-3 `Authorization` header — omit it when `AUTH_DISABLED=true`.

---

## Workspaces

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/workspaces` | `{name}` → workspace | creator becomes `owner` |
| GET | `/workspaces` | → `[workspaces]` | caller's memberships only |
| POST | `/workspaces/{id}/invite` | `{email, role}` → `{invite_id}` | owner only |
| GET | `/workspaces/{id}/members` | → `[{user, role}]` | member only |

### `POST /workspaces`
**Auth:** any authenticated user.

Request:
```json
{ "name": "Client X" }
```
`201 Created`:
```json
{
  "id": "8a1f0c2e-4b7d-4e9a-b3c1-2d5e6f7a8b9c",
  "name": "Client X",
  "spend_cap_usd": 100.00,
  "role": "owner",
  "created_at": "2026-08-24T12:00:00Z"
}
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | `name` missing/blank |
| 401 | `UNAUTHORIZED` | no/invalid JWT |

```json
{ "error": { "code": "VALIDATION", "message": "name is required" } }
```
```bash
curl -X POST http://localhost:8000/api/v1/workspaces \
  -H "Authorization: Bearer $LOGTO_JWT" \
  -H "Content-Type: application/json" \
  -d '{"name":"Client X"}'
```

### `GET /workspaces`
**Auth:** any authenticated user (own memberships only).

`200 OK`:
```json
[
  {
    "id": "8a1f0c2e-4b7d-4e9a-b3c1-2d5e6f7a8b9c",
    "name": "Client X",
    "role": "owner",
    "spend_cap_usd": 100.00,
    "created_at": "2026-08-24T12:00:00Z"
  }
]
```
Errors: 401 `UNAUTHORIZED` only.
```bash
curl http://localhost:8000/api/v1/workspaces \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `POST /workspaces/{id}/invite`
**Auth:** `owner` of `{id}`.

Request:
```json
{ "email": "teammate@example.com", "role": "viewer" }
```
`201 Created`:
```json
{ "invite_id": "3c4d5e6f-7a8b-9c0d-1e2f-3a4b5c6d7e8f" }
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | bad email; `role` not in `owner/analyst/viewer` |
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 403 | `FORBIDDEN` | caller is member but not owner |
| 404 | `NOT_FOUND` | workspace missing or caller not a member |
| 409 | `CONFLICT` | email already a member |

```json
{ "error": { "code": "FORBIDDEN", "message": "only the workspace owner can invite members" } }
```
```bash
curl -X POST http://localhost:8000/api/v1/workspaces/8a1f0c2e-4b7d-4e9a-b3c1-2d5e6f7a8b9c/invite \
  -H "Authorization: Bearer $LOGTO_JWT" \
  -H "Content-Type: application/json" \
  -d '{"email":"teammate@example.com","role":"viewer"}'
```

### `GET /workspaces/{id}/members`
**Auth:** `member+` of `{id}`.

`200 OK`:
```json
[
  { "user": { "id": "7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c", "email": "owner@example.com", "display_name": "A" }, "role": "owner" },
  { "user": { "id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d", "email": "teammate@example.com", "display_name": "B" }, "role": "viewer" }
]
```
Errors:

| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 404 | `NOT_FOUND` | workspace missing or caller not a member |

```bash
curl http://localhost:8000/api/v1/workspaces/8a1f0c2e-4b7d-4e9a-b3c1-2d5e6f7a8b9c/members \
  -H "Authorization: Bearer $LOGTO_JWT"
```

## Projects

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/workspaces/{wid}/projects` | `{name, question, spend_cap_usd}` → project | analyst+ |
| GET | `/workspaces/{wid}/projects` | → `[{id, name, status, cost_to_date, confidence}]` | cards for home screen |
| GET | `/projects/{id}` | → project detail incl. status history | |
| POST | `/projects/{id}/seed` | multipart file (PDF/MD/TXT) → `{seed_id, version, preview_text}` | ≤10 MB; text extracted server-side |

### `POST /workspaces/{wid}/projects`
**Auth:** `analyst+` of `{wid}`. Workspace ceiling check applies (P3-T5).

Request:
```json
{ "name": "Nothing Phone 3 test", "question": "Should we launch at ₹34,999?", "spend_cap_usd": 25.00 }
```
`201 Created`:
```json
{
  "id": "1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6",
  "workspace_id": "8a1f0c2e-4b7d-4e9a-b3c1-2d5e6f7a8b9c",
  "name": "Nothing Phone 3 test",
  "question": "Should we launch at ₹34,999?",
  "status": "draft",
  "spend_cap_usd": 25.00,
  "created_by": "7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c",
  "created_at": "2026-08-24T12:01:00Z"
}
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | missing field; `spend_cap_usd` ≤ 0 |
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 402 | `BUDGET_EXCEEDED` | cap would exceed remaining workspace budget (mt-03 step 11) |
| 403 | `FORBIDDEN` | caller is `viewer` |
| 404 | `NOT_FOUND` | workspace missing or caller not a member |

```json
{ "error": { "code": "BUDGET_EXCEEDED", "message": "project cap 25.00 exceeds remaining workspace budget 12.50" } }
```
```bash
curl -X POST http://localhost:8000/api/v1/workspaces/8a1f0c2e-4b7d-4e9a-b3c1-2d5e6f7a8b9c/projects \
  -H "Authorization: Bearer $LOGTO_JWT" \
  -H "Content-Type: application/json" \
  -d '{"name":"Nothing Phone 3 test","question":"Should we launch at ₹34,999?","spend_cap_usd":25.00}'
```

### `GET /workspaces/{wid}/projects`
**Auth:** `member+` of `{wid}`.

`200 OK` (card shape for the home screen; `confidence` is null until a verdict exists):
```json
[
  { "id": "1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6", "name": "Nothing Phone 3 test", "status": "grounded", "cost_to_date": 0.83, "confidence": null }
]
```
Errors: 401 `UNAUTHORIZED`; 404 `NOT_FOUND` (workspace missing or not a member).
```bash
curl http://localhost:8000/api/v1/workspaces/8a1f0c2e-4b7d-4e9a-b3c1-2d5e6f7a8b9c/projects \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `GET /projects/{id}`
**Auth:** `member+` of the owning workspace.

`200 OK`:
```json
{
  "id": "1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6",
  "workspace_id": "8a1f0c2e-4b7d-4e9a-b3c1-2d5e6f7a8b9c",
  "name": "Nothing Phone 3 test",
  "question": "Should we launch at ₹34,999?",
  "status": "grounded",
  "spend_cap_usd": 25.00,
  "cost_to_date": 0.83,
  "latest_seed_version": 1,
  "status_history": [
    { "from": null, "to": "draft", "at": "2026-08-24T12:01:00Z" },
    { "from": "draft", "to": "grounding", "at": "2026-08-24T12:05:11Z" },
    { "from": "grounding", "to": "grounded", "at": "2026-08-24T12:31:47Z" }
  ],
  "created_by": "7f8a9b0c-1d2e-3f4a-5b6c-7d8e9f0a1b2c",
  "created_at": "2026-08-24T12:01:00Z"
}
```
Errors: 401 `UNAUTHORIZED`; 404 `NOT_FOUND` (project missing or not visible to caller).
```bash
curl http://localhost:8000/api/v1/projects/1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6 \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `POST /projects/{id}/seed`
**Auth:** `analyst+`. Multipart upload, field name `file`; PDF/MD/TXT, ≤10 MB; text extracted server-side.

`201 Created`:
```json
{
  "seed_id": "0b1c2d3e-4f5a-6b7c-8d9e-0f1a2b3c4d5e",
  "version": 1,
  "preview_text": "Launch memo: Nothing Phone 3 targets the premium-mid segment..."
}
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | unsupported content type; unextractable text |
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 403 | `FORBIDDEN` | caller is `viewer` |
| 404 | `NOT_FOUND` | project missing or not visible |
| 413 | `VALIDATION` | file > 10 MB |

```json
{ "error": { "code": "VALIDATION", "message": "seed file exceeds 10 MB limit" } }
```
```bash
curl -X POST http://localhost:8000/api/v1/projects/1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6/seed \
  -H "Authorization: Bearer $LOGTO_JWT" \
  -F file=@/path/to/seed.md
```

## Grounding

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/projects/{id}/grounding` | `{keywords[], sources[], region, languages[], time_window_days}` → `{job_id}` | sets status `grounding`; calls adapter POST /collect |
| GET | `/grounding/{job_id}` | → status + progress (adapter shape) | polled by Grounding Console |
| GET | `/grounding/{job_id}/items?source=&offset=&limit=` | → paged items | sample quotes for console |
| POST | `/grounding/{job_id}/analyze` | → `{report_id}` | adapter POST /analyze |
| GET | `/reports/baseline/{report_id}` | → baseline report, **only if citation-verified** | else `422 INSUFFICIENT_GROUNDING` |

### `POST /projects/{id}/grounding`
**Auth:** `analyst+`. Sets project status `grounding`; forwards to adapter `POST /collect` (adapter-contract.md §1).

Request:
```json
{
  "keywords": ["Nothing Phone 3"],
  "sources": ["reddit", "youtube"],
  "region": "global",
  "languages": ["en"],
  "time_window_days": 30
}
```
`202 Accepted`:
```json
{ "job_id": "5f6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c" }
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | empty `keywords`/`sources`; unsupported source; `time_window_days` out of range |
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 403 | `FORBIDDEN` | caller is `viewer` |
| 404 | `NOT_FOUND` | project missing or not visible |
| 409 | `CONFLICT` | a grounding job is already running for this project |

```json
{ "error": { "code": "CONFLICT", "message": "grounding job 5f6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c already running" } }
```
```bash
curl -X POST http://localhost:8000/api/v1/projects/1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6/grounding \
  -H "Authorization: Bearer $LOGTO_JWT" \
  -H "Content-Type: application/json" \
  -d '{"keywords":["Nothing Phone 3"],"sources":["reddit","youtube"],"region":"global","languages":["en"],"time_window_days":30}'
```

### `GET /grounding/{job_id}`
**Auth:** `member+` of the job's workspace. Proxy of adapter `GET /collect/{job_id}/status` (shape identical).

`200 OK`:
```json
{
  "job_id": "5f6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c",
  "status": "running",
  "progress": {
    "reddit":  { "collected": 320, "target": 500 },
    "youtube": { "collected": 150, "target": 500 }
  },
  "error": null
}
```
Errors: 401 `UNAUTHORIZED`; 404 `NOT_FOUND` (job missing or not visible).
```bash
curl http://localhost:8000/api/v1/grounding/5f6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `GET /grounding/{job_id}/items`
**Auth:** `member+`. Query: `source` (optional filter), `offset` (default 0), `limit` (default 50, ≤200 — behavioral requirement 5).

`200 OK` (PII-stripped adapter item shape):
```json
{
  "items": [
    {
      "item_id": "b3d9f2c1a8e4…sha256…",
      "source": "reddit",
      "url": "https://www.reddit.com/r/Android/comments/abc123/",
      "published_at": "2026-08-20T09:14:00Z",
      "text": "The camera bump alone makes this a no for me at that price.",
      "language": "en",
      "metrics": { "score": 123, "comments": 45 },
      "region": null
    }
  ],
  "total": 320,
  "offset": 0,
  "limit": 50
}
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | `limit` > 200; negative `offset` |
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 404 | `NOT_FOUND` | job missing or not visible |

```json
{ "error": { "code": "VALIDATION", "message": "limit must be ≤ 200" } }
```
```bash
curl "http://localhost:8000/api/v1/grounding/5f6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c/items?source=reddit&offset=0&limit=50" \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `POST /grounding/{job_id}/analyze`
**Auth:** `analyst+`. Forwards to adapter `POST /analyze/{job_id}`; runs on `report-model` via LiteLLM (LLM spend recorded per behavioral requirement 4).

`202 Accepted`:
```json
{ "report_id": "7a8b9c0d-1e2f-3a4b-5c6d-7e8f9a0b1c2d" }
```
Errors:

| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 403 | `FORBIDDEN` | caller is `viewer` |
| 404 | `NOT_FOUND` | job missing or not visible |
| 409 | `CONFLICT` | job not `done` yet, or analysis already running |

```json
{ "error": { "code": "CONFLICT", "message": "grounding job is still running" } }
```
```bash
curl -X POST http://localhost:8000/api/v1/grounding/5f6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c/analyze \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `GET /reports/baseline/{report_id}`
**Auth:** `member+`. Served **only if citation-verified** (adapter §1 hard rule; mt-01 step 10).

`200 OK` while running:
```json
{ "report_id": "7a8b9c0d-1e2f-3a4b-5c6d-7e8f9a0b1c2d", "status": "running" }
```
`200 OK` when done and verified:
```json
{
  "report_id": "7a8b9c0d-1e2f-3a4b-5c6d-7e8f9a0b1c2d",
  "status": "done",
  "verified": true,
  "summary": {
    "sentiment": { "positive": 0.41, "neutral": 0.33, "negative": 0.26 },
    "themes": [
      {
        "label": "price concerns",
        "item_ids": ["b3d9f2c1a8e4…", "c7e1a5b2d9f3…"],
        "claim": "Users say the launch price is too high for the camera hardware offered",
        "citation_item_ids": ["b3d9f2c1a8e4…"]
      }
    ],
    "key_entities": ["Nothing Phone 3", "camera bump", "₹34,999"]
  }
}
```
Errors:

| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 404 | `NOT_FOUND` | report missing or not visible |
| 422 | `INSUFFICIENT_GROUNDING` | report done but citation check failed (`verified=false`) |

```json
{ "error": { "code": "INSUFFICIENT_GROUNDING", "message": "baseline report failed citation verification; rerun analyze" } }
```
```bash
curl http://localhost:8000/api/v1/reports/baseline/7a8b9c0d-1e2f-3a4b-5c6d-7e8f9a0b1c2d \
  -H "Authorization: Bearer $LOGTO_JWT"
```

## Personas

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/projects/{id}/persona-panel/generate` | → `{panel_id, version}` | `report-model` via LiteLLM; input = verified baseline report + item samples |
| GET | `/projects/{id}/persona-panel` | → latest panel | |
| PUT | `/persona-panels/{panel_id}` | `{panel}` → new version | proportions renormalized; stored as new row |

### `POST /projects/{id}/persona-panel/generate`
**Auth:** `analyst+`. Synchronous long call on `report-model` (expect tens of seconds); input = verified baseline report + item samples; LLM spend recorded (requirement 4).

`201 Created`:
```json
{ "panel_id": "2b3c4d5e-6f7a-8b9c-0d1e-2f3a4b5c6d7e", "version": 1 }
```
Errors:

| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 403 | `FORBIDDEN` | caller is `viewer` |
| 404 | `NOT_FOUND` | project missing or not visible |
| 409 | `CONFLICT` | generation already in progress |
| 422 | `INSUFFICIENT_GROUNDING` | no verified baseline report for the latest grounding job |

```json
{ "error": { "code": "INSUFFICIENT_GROUNDING", "message": "persona panel requires a citation-verified baseline report" } }
```
```bash
curl -X POST http://localhost:8000/api/v1/projects/1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6/persona-panel/generate \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `GET /projects/{id}/persona-panel`
**Auth:** `member+`. Returns the latest (highest-version) panel; `edited_by` is null for auto-generated panels.

`200 OK` (panel shape per adapter §2):
```json
{
  "panel_id": "2b3c4d5e-6f7a-8b9c-0d1e-2f3a4b5c6d7e",
  "project_id": "1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6",
  "version": 1,
  "edited_by": null,
  "created_at": "2026-08-24T12:40:03Z",
  "panel": {
    "archetypes": [
      {
        "name": "Value-conscious Android enthusiast",
        "proportion": 0.35,
        "stance": "skeptical",
        "description": "Follows every launch; compares specs-per-rupee against last-gen flagships",
        "language_samples": ["The camera bump alone makes this a no for me at that price."],
        "llm_model": "swarm-model"
      }
    ]
  }
}
```
Errors: 401 `UNAUTHORIZED`; 404 `NOT_FOUND` (project missing/not visible, or no panel generated yet).
```bash
curl http://localhost:8000/api/v1/projects/1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6/persona-panel \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `PUT /persona-panels/{panel_id}`
**Auth:** `analyst+`. Full-panel replacement; stored as a **new version row** (old versions untouched, mt-02 step 4); proportions renormalized to 1.0.

Request:
```json
{
  "panel": {
    "archetypes": [
      {
        "name": "Value-conscious Android enthusiast",
        "proportion": 0.40,
        "stance": "skeptical",
        "description": "Follows every launch; compares specs-per-rupee against last-gen flagships",
        "language_samples": ["The camera bump alone makes this a no for me at that price."],
        "llm_model": "swarm-model"
      }
    ]
  }
}
```
`201 Created`:
```json
{ "panel_id": "2b3c4d5e-6f7a-8b9c-0d1e-2f3a4b5c6d7e", "version": 2 }
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | empty `archetypes`; archetype missing required fields; `stance` outside enum |
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 403 | `FORBIDDEN` | caller is `viewer` |
| 404 | `NOT_FOUND` | panel missing or not visible |

```bash
curl -X PUT http://localhost:8000/api/v1/persona-panels/2b3c4d5e-6f7a-8b9c-0d1e-2f3a4b5c6d7e \
  -H "Authorization: Bearer $LOGTO_JWT" \
  -H "Content-Type: application/json" \
  -d '{"panel":{"archetypes":[{"name":"Value-conscious Android enthusiast","proportion":0.40,"stance":"skeptical","description":"Follows every launch; compares specs-per-rupee","language_samples":["The camera bump alone makes this a no for me at that price."],"llm_model":"swarm-model"}]}}'
```

## Simulation

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| POST | `/projects/{id}/simulations` | `{variants: [{label, seed_patch_md}]}` → `{simulation_id}` | runs all pre-flight checks (data-model invariants 2–3); launches adaptive ensemble per adapter §3 |
| GET | `/simulations/{sid}` | → status (adapter shape + cost_to_date) | |
| GET | `/simulations/{sid}/results` | → per-variant aggregated results | only when ensemble complete |
| POST | `/simulations/{sid}/ask` | `{run_id, agent_id, question}` → answer | pass-through to adapter; `simulated: true` always in response |
| WS | `/ws/simulations/{sid}` | stream: `{type: "round", run_id, round, posts: [...]}` | live theater feed; proxy of MiroShark events |

### `POST /projects/{id}/simulations`
**Auth:** `analyst+`. Pre-flight: data-model invariants 2 (verified baseline) and 3 (budget) run **before** anything is sent to MiroShark. ≤3 variants (MVP). On success project status → `simulating`.

Request:
```json
{
  "variants": [
    { "label": "Price ₹29,999", "seed_patch_md": "The price is ₹29,999." },
    { "label": "Price ₹39,999", "seed_patch_md": "The price is ₹39,999." }
  ]
}
```
`202 Accepted`:
```json
{ "simulation_id": "4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a" }
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | no variants; >3 variants; blank `label`/`seed_patch_md` |
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 402 | `BUDGET_EXCEEDED` | `cost_to_date + estimate > spend_cap_usd` → project status `halted_budget` (requirement 2, mt-02 step 13) |
| 403 | `FORBIDDEN` | caller is `viewer` |
| 404 | `NOT_FOUND` | project missing or not visible |
| 409 | `CONFLICT` | another simulation already running for this project |
| 422 | `INSUFFICIENT_GROUNDING` | no verified baseline report (requirement 1, mt-02 step 6 — no exceptions) |

```json
{ "error": { "code": "BUDGET_EXCEEDED", "message": "estimated cost 3.10 + spent 24.97 exceeds spend_cap_usd 25.00; project halted_budget" } }
```
```bash
curl -X POST http://localhost:8000/api/v1/projects/1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6/simulations \
  -H "Authorization: Bearer $LOGTO_JWT" \
  -H "Content-Type: application/json" \
  -d '{"variants":[{"label":"Price ₹29,999","seed_patch_md":"The price is ₹29,999."},{"label":"Price ₹39,999","seed_patch_md":"The price is ₹39,999."}]}'
```

### `GET /simulations/{sid}`
**Auth:** `member+`. Adapter status shape (adapter §2) plus core-side `cost_to_date`.

`200 OK`:
```json
{
  "simulation_id": "4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a",
  "project_id": "1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6",
  "status": "running",
  "cost_to_date": 1.42,
  "runs": [
    {
      "run_id": "6e7f8a9b-0c1d-2e3f-4a5b-6c7d8e9f0a1b",
      "variant_id": "9a0b1c2d-3e4f-5a6b-7c8d-9e0f1a2b3c4d",
      "variant_label": "Price ₹29,999",
      "ensemble_index": 0,
      "status": "running",
      "current_round": 12,
      "total_rounds": 18,
      "early_stopped": false
    }
  ]
}
```
Errors: 401 `UNAUTHORIZED`; 404 `NOT_FOUND` (simulation missing or not visible).
```bash
curl http://localhost:8000/api/v1/simulations/4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `GET /simulations/{sid}/results`
**Auth:** `member+`. Available **only when the ensemble is complete** (adapter §3: converged, or 7 runs → `NO_CONSENSUS`). Every `quote_refs` entry resolves to a simulated post and is labeled simulated — never presented as real.

`200 OK`:
```json
{
  "simulation_id": "4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a",
  "variants": [
    {
      "variant_id": "9a0b1c2d-3e4f-5a6b-7c8d-9e0f1a2b3c4d",
      "label": "Price ₹29,999",
      "ensemble_runs": 3,
      "agreement": 0.83,
      "converged": true,
      "runs": [
        {
          "run_id": "6e7f8a9b-0c1d-2e3f-4a5b-6c7d8e9f0a1b",
          "final_sentiment": { "positive": 0.5, "neutral": 0.2, "negative": 0.3 },
          "stance_by_archetype": [
            { "archetype": "Value-conscious Android enthusiast", "support": 0.6, "oppose": 0.3, "neutral": 0.1 }
          ],
          "top_objections": [
            { "theme": "battery life", "quote_refs": ["agent_012:7:post_88"] }
          ],
          "narratives": [
            { "label": "worth-it-at-this-price", "momentum": "rising" }
          ]
        }
      ]
    }
  ]
}
```
Errors:

| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 404 | `NOT_FOUND` | simulation missing or not visible |
| 409 | `CONFLICT` | ensemble still running — results not final |

```json
{ "error": { "code": "CONFLICT", "message": "ensemble incomplete: 2 of 3 runs done for variant 9a0b1c2d" } }
```
```bash
curl http://localhost:8000/api/v1/simulations/4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a/results \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `POST /simulations/{sid}/ask`
**Auth:** `member+`. Pass-through to adapter `POST /simulations/{sid}/ask`. **`simulated: true` is always present** — UI fails closed without it (websocket-protocol.md hard rule 1). Each ask is an LLM call and records into `cost_ledger` (requirement 4).

Request:
```json
{ "run_id": "6e7f8a9b-0c1d-2e3f-4a5b-6c7d8e9f0a1b", "agent_id": "agent_012", "question": "Why did you oppose the price?" }
```
`200 OK`:
```json
{
  "answer": "At ₹39,999 the battery anxiety isn't justified by the camera upgrade — I said so in round 7.",
  "simulated": true,
  "run_id": "6e7f8a9b-0c1d-2e3f-4a5b-6c7d8e9f0a1b",
  "round_context": [3, 7]
}
```
Errors:

| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION` | blank `question` |
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 404 | `NOT_FOUND` | simulation/run/agent missing or not visible |

```bash
curl -X POST http://localhost:8000/api/v1/simulations/4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a/ask \
  -H "Authorization: Bearer $LOGTO_JWT" \
  -H "Content-Type: application/json" \
  -d '{"run_id":"6e7f8a9b-0c1d-2e3f-4a5b-6c7d8e9f0a1b","agent_id":"agent_012","question":"Why did you oppose the price?"}'
```

### `WS /ws/simulations/{sid}`
**Auth:** `member+`. Phase 4+: one-time ticket from `POST /simulations/{sid}/ws-ticket` passed as `?ticket=` (never the JWT in the URL); phases 0–2 connect directly. Full event catalog, reconnect/resume, backpressure and rate-limit rules: **contracts/websocket-protocol.md** (source of truth — not duplicated here).

Server → client example event:
```json
{ "type": "round", "run_id": "6e7f8a9b-0c1d-2e3f-4a5b-6c7d8e9f0a1b", "round": 7, "total_rounds": 18,
  "posts": [{ "post_id": "post_88", "agent_id": "agent_012", "archetype": "Value-conscious Android enthusiast",
              "text": "Battery life kills it for me.", "stance": "oppose",
              "reply_to": null, "simulated": true }],
  "sentiment_tick": { "positive": 0.44, "neutral": 0.31, "negative": 0.25 } }
```
Errors: auth/policy failures arrive as `{"type":"error","code":"UNAUTHORIZED","message":"..."}` then the socket closes (codes per websocket-protocol.md).

Copy-pasteable connect (curl cannot speak the WebSocket frame protocol — use `websocat`/`wscat`):
```bash
websocat "ws://localhost:8000/api/v1/ws/simulations/4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a?ticket=$WS_TICKET"
```

## Verdicts & cost

| Method | Path | Body → Response | Notes |
|---|---|---|---|
| GET | `/projects/{id}/verdict` | → latest verdict (incl. `NO_CONSENSUS` state) | one-page payload for verdict UI |
| GET | `/projects/{id}/costs` | → `{total, by_component: {...}, cap, remaining}` | from `cost_ledger` |

### `GET /projects/{id}/verdict`
**Auth:** `member+`. Latest verdict for the project; one-page payload for the verdict UI. `confidence` derives from `agreement_score` (data-model invariant 4).

`200 OK` (converged):
```json
{
  "id": "5c6d7e8f-9a0b-1c2d-3e4f-5a6b7c8d9e0f",
  "project_id": "1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6",
  "simulation_id": "4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a",
  "outcome": "verdict",
  "agreement_score": 0.833,
  "confidence": "medium",
  "ranked_variants": [
    { "variant_id": "9a0b1c2d-3e4f-5a6b-7c8d-9e0f1a2b3c4d", "label": "Price ₹29,999", "direction": "support", "support": 0.62 }
  ],
  "objections": [
    {
      "theme": "battery life",
      "quotes": [
        { "text": "Battery life kills it for me.", "origin": "sim", "ref": "agent_012:7:post_88" },
        { "text": "The camera bump alone makes this a no for me at that price.", "origin": "real", "ref": "b3d9f2c1a8e4…" }
      ]
    }
  ],
  "risk_flags": ["thin agreement on objection themes for variant Price ₹39,999"],
  "report_markdown": "# Verdict: launch at ₹29,999\n…",
  "created_at": "2026-08-24T14:05:00Z"
}
```
`200 OK` (`NO_CONSENSUS` — never forced; ranked data must not be presented, mt-02 step 9):
```json
{
  "id": "6d7e8f9a-0b1c-2d3e-4f5a-6b7c8d9e0f1a",
  "project_id": "1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6",
  "simulation_id": "4d5e6f7a-8b9c-0d1e-2f3a-4b5c6d7e8f9a",
  "outcome": "no_consensus",
  "agreement_score": 0.667,
  "confidence": null,
  "ranked_variants": null,
  "objections": null,
  "risk_flags": ["7 ensemble runs without convergence", "direction split: 4 support / 3 oppose on Price ₹29,999"],
  "report_markdown": "# No consensus\nRuns diverged on…",
  "created_at": "2026-08-24T15:40:00Z"
}
```
Errors:

| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no/invalid JWT |
| 404 | `NOT_FOUND` | project missing/not visible, **or no verdict issued yet** (still simulating) |

```json
{ "error": { "code": "NOT_FOUND", "message": "no verdict yet for this project" } }
```
```bash
curl http://localhost:8000/api/v1/projects/1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6/verdict \
  -H "Authorization: Bearer $LOGTO_JWT"
```

### `GET /projects/{id}/costs`
**Auth:** `member+`. Aggregated from `cost_ledger`; `by_component` keys are the ledger's component enum.

`200 OK`:
```json
{
  "total": 2.31,
  "by_component": {
    "grounding": 0.12,
    "analysis": 0.41,
    "simulation": 1.42,
    "verdict": 0.36,
    "report": 0.00
  },
  "cap": 25.00,
  "remaining": 22.69
}
```
Errors: 401 `UNAUTHORIZED`; 404 `NOT_FOUND` (project missing or not visible).
```bash
curl http://localhost:8000/api/v1/projects/1c2d3e4f-5a6b-7c8d-9e0f-a1b2c3d4e5f6/costs \
  -H "Authorization: Bearer $LOGTO_JWT"
```

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
