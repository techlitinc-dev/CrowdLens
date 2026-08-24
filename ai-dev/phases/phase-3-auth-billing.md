# Phase 3 — Auth, Workspaces & Budget Enforcement

**Goal:** real multi-tenant accounts and hard spend caps. After this phase the MVP cut (PBR §12) is complete. **Gate:** `manual-tests/mt-03-auth-billing.md` all green.

## Prerequisites
- Phase 2 green. Human creates a self-hosted Logto instance (add to compose per official Logto self-host docs) and puts `LOGTO_ENDPOINT`, `LOGTO_APP_ID`, `LOGTO_APP_SECRET`, `LOGTO_M2M_*` in `.env`.

## Tasks

### P3-T1 — Logto service + app registration
Add `logto` (+ its postgres) to compose. Human does the in-console app registration (console UI — agent can't); agent writes the exact setup steps into `infra/logto-setup.md` as a checklist for the human: create Traditional Web app, set redirect URIs (`http://localhost:5173/auth/callback`), enable email + Google/GitHub connectors, create org roles `owner`/`analyst`/`viewer`.
**Done when:** human confirms each checklist item in mt-03 step 1.

### P3-T2 — Backend JWT verification + RBAC
core-api middleware: validate Logto JWTs (JWKS from Logto endpoint), resolve `users`/`workspace_members` rows (auto-provision user on first login), enforce role matrix:

| Action | owner | analyst | viewer |
|---|---|---|---|
| create/edit project, launch grounding/sim | ✓ | ✓ | ✗ |
| upload seed, edit persona panel | ✓ | ✓ | ✗ |
| invite members, change caps, billing | ✓ | ✗ | ✗ |
| view everything | ✓ | ✓ | ✓ |

`AUTH_DISABLED=true` keeps working for dev/test. Enforcement is a dependency (`require_role`), applied on every endpoint from `contracts/api-surface.md`.
**Done when:** integration tests: viewer gets 403 on `POST /projects`, 200 on GETs; owner-only invite endpoint; unknown JWT → 401.

### P3-T3 — Frontend auth flow
Logto React SDK: login button, callback route, silent token refresh, JWT attached to every API call, 401 → redirect to login. Role-aware UI: viewers don't see "New Project" buttons (server still enforces — UI hiding is convenience only).
**Done when:** mt-03 step 3 browser flow passes.

### P3-T4 — Per-project LiteLLM virtual keys + hard caps
On project creation: core-api calls LiteLLM `/key/generate` with `max_budget = project.spend_cap_usd` → stores key in `projects.litellm_key`. Every adapter LLM call uses that key. On `POST /projects` with a cap change → update key budget. LiteLLM returning 402 budget errors → project `halted_budget`, surfaced in API.
**Done when:** integration test with LiteLLM mock: cap $1, simulated spend past it → LiteLLM rejects → project halted; and mt-03 step 5 with a real tiny cap.

### P3-T5 — Workspace cap + audit log
Workspace cap = sum ceiling across its projects (validate on project create/update). Create the `audit_events` table **exactly as defined in `contracts/data-model.md` → Later-phase tables** (do not redefine it here). Write events for: login, invite, project create, grounding launch, simulation launch, verdict view/export, cap change. `GET /workspaces/{id}/audit` (owner only, paged).
**Done when:** migration clean; performing each action writes the matching row; endpoint serves it.

## Stop conditions
- Logto self-host version behaves differently than the checklist assumes → stop, report exact mismatch.
- LiteLLM virtual-key budget API changed → stop, link the upstream docs, ask.
- Never weaken a role check to make a test pass. Report instead.

## Manual test
`../manual-tests/mt-03-auth-billing.md` — all green = MVP complete per PBR §12.
