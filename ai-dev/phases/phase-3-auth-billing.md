# Phase 3 — Auth, Workspaces & Budget Enforcement

**Goal:** real multi-tenant accounts and hard spend caps. After this phase the MVP cut (PBR §12) is complete. **Gate:** `manual-tests/mt-03-auth-billing.md` all green.

## Prerequisites
- Phase 2 green. Human creates a self-hosted Logto instance (add to compose per official Logto self-host docs) and puts `LOGTO_ENDPOINT`, `LOGTO_APP_ID`, `LOGTO_APP_SECRET`, `LOGTO_M2M_*` in `.env`.

## Tasks

### P3-T1 — Logto service + app registration
Add `logto` (+ its postgres) to compose. Human does the in-console app registration (console UI — agent can't); agent writes the exact setup steps into `infra/logto-setup.md` as a checklist for the human: create Traditional Web app, set redirect URIs (`http://localhost:5173/auth/callback`), enable email + Google/GitHub connectors, create org roles `owner`/`analyst`/`viewer`.
**Done when:** human confirms each checklist item in mt-03 step 1.

**Files:**
- `infra/docker-compose.yml` — add `logto` + `logto-postgres` services per the official Logto self-host compose (ports 3001 app / 3002 admin; check conflicts before committing them, stop condition below applies).
- `infra/logto-setup.md` — the human checklist (numbered, one checkbox per console action).
- `.env.example` — add `LOGTO_ENDPOINT`, `LOGTO_APP_ID`, `LOGTO_APP_SECRET`, `LOGTO_M2M_CLIENT_ID`, `LOGTO_M2M_CLIENT_SECRET` (no values).

**Tests:** `tests/integration/test_logto_health.py::test_oidc_discovery_reachable` — GET `{LOGTO_ENDPOINT}/oidc/.well-known/openid-configuration` → 200 and `jwks_uri` present (skipped when Logto env unset, so CI without Logto stays green).

**Edge cases:** Logto DB init race on first `up` (its postgres must be healthy first); admin console port ≠ app port — checklist must state both explicitly; org roles must be created in the Logto **organization** template, not per-app, or role claims won't appear in JWTs (P3-T2 reads them).

**Cheap-LLM failure modes:** none — no LLM in this path.

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

**Files:**
- `app/auth/jwks.py` — JWKS fetch + TTL cache from the Logto OIDC discovery `jwks_uri`.
- `app/auth/deps.py` — FastAPI dependencies (below).
- `app/auth/provision.py` — first-login auto-provisioning (`users` row keyed by `logto_user_id`, workspace membership from Logto org membership).
- `app/main.py` + every router in `app/routers/` — wire the dependencies; no router keeps an unguarded write path.

**Key functions:**
- `async def get_current_user(request: Request) -> User` — 401 `UNAUTHORIZED` on missing/expired/wrong-audience token; when `AUTH_DISABLED=true` returns the synthetic dev owner (api-surface.md RBAC note).
- `def require_role(*roles: str) -> Callable` — dependency factory; 403 `FORBIDDEN` when the caller's workspace role isn't in `roles`.
- `async def _jwks(force_refresh: bool = False) -> dict` — cached; on unknown `kid` refetch once, then fail.

**Tests** (`tests/integration/test_auth_rbac.py`): `test_missing_token_401`, `test_garbage_jwt_401`, `test_expired_token_401`, `test_viewer_post_projects_403`, `test_viewer_get_endpoints_200`, `test_invite_owner_only`, `test_analyst_cannot_change_caps`, `test_first_login_auto_provisions_user`, `test_auth_disabled_still_acts_as_owner`, `test_workspace_isolation_no_cross_reads`.

**Edge cases:** JWKS key rotation mid-session (unknown `kid` → single forced refetch, not a crash); small clock-skew leeway on `exp`; a user removed from the org but holding an unexpired JWT — membership is re-resolved from the DB per request, not trusted from the token; role change doesn't take effect until the claims/DB row update — document, don't cache forever.

**Cheap-LLM failure modes:** none — no LLM in this path.

### P3-T3 — Frontend auth flow
Logto React SDK: login button, callback route, silent token refresh, JWT attached to every API call, 401 → redirect to login. Role-aware UI: viewers don't see "New Project" buttons (server still enforces — UI hiding is convenience only).
**Done when:** mt-03 step 3 browser flow passes.

**Files:**
- `frontend/src/main.tsx` — `LogtoProvider` config (endpoint, appId, redirect URI per P3-T1).
- `frontend/src/routes/auth/CallbackPage.tsx` — handles `/auth/callback`.
- `frontend/src/api/client.ts` — the single fetch wrapper: attaches the access token to every call, on 401 triggers re-login redirect (never silently retries with a dead token).
- `frontend/src/hooks/useRole.ts` — current workspace role for UI gating; feeds `NewProjectButton`/launch-button visibility.

**Key functions:**
- `async function getApiToken(): Promise<string>` — wraps the Logto React SDK's token getter (per official Logto React SDK docs); throws → caller redirects to login.
- `function useRole(workspaceId: string): 'owner' | 'analyst' | 'viewer' | undefined` — from `GET /workspaces/{id}/members` via TanStack Query.

**Tests:** no frontend unit harness exists yet — gate is `npm run build` clean + mt-03 steps 3–7 in the browser (incl. the incognito second account).

**Edge cases:** silent refresh failure (offline / revoked session) → redirect, not an infinite spinner; two accounts side-by-side (incognito) — TanStack Query cache must be per-session, no cross-window leakage; JWT expiring mid-long-poll on the grounding console → one retry after refresh, then honest error.

**Cheap-LLM failure modes:** none — no LLM in this path.

### P3-T4 — Per-project LiteLLM virtual keys + hard caps
On project creation: core-api calls LiteLLM `/key/generate` with `max_budget = project.spend_cap_usd` → stores key in `projects.litellm_key`. Every adapter LLM call uses that key. On `POST /projects` with a cap change → update key budget. LiteLLM returning 402 budget errors → project `halted_budget`, surfaced in API.
**Done when:** integration test with LiteLLM mock: cap $1, simulated spend past it → LiteLLM rejects → project halted; and mt-03 step 5 with a real tiny cap.

**Files:**
- `app/services/litellm_keys.py` — all LiteLLM Admin API calls (per official LiteLLM proxy docs for `/key/generate` + key update).
- `app/routers/projects.py` — key generation hooked into project create; budget update hooked into cap change.
- `app/services/llm.py` (or wherever LLM calls are issued) — every call carries the project's virtual key; budget-reject responses mapped to `halted_budget`.

**Key functions:**
- `async def generate_project_key(project_id: UUID, max_budget_usd: Decimal) -> str` — returns the virtual key; stored in `projects.litellm_key`.
- `async def update_key_budget(key: str, max_budget_usd: Decimal) -> None` — used on cap raise/lower.
- `def is_budget_reject(status: int, body: dict) -> bool` — recognizes LiteLLM's budget-exceeded error; caller sets project status `halted_budget` and writes the `budget_halt` audit event.

**Tests** (`tests/integration/test_budget_caps.py`, LiteLLM mocked with `respx`): `test_project_create_generates_virtual_key`, `test_cap_change_updates_key_budget`, `test_budget_reject_halts_project`, `test_halt_reason_surfaced_on_project_get`, `test_no_llm_call_ever_uses_master_key`.

**Edge cases:** LiteLLM unreachable at project create → project row created without key, key generated lazily+retried, never fall back to `LITELLM_MASTER_KEY` (that bypasses all caps — P0); cap lowered below current spend → next call rejects, halt is honest not silent; LiteLLM budget checks are per-request, so a burst can overshoot by one call — `cost_ledger` is the reconciling record; key budget update failing after the DB cap already changed → return 502 and don't report the cap as changed.

**Cheap-LLM failure modes:** swarm-model responses never influence budget logic — spend comes from LiteLLM's accounting, never from model self-reports.

### P3-T5 — Workspace cap + audit log
Workspace cap = sum ceiling across its projects (validate on project create/update). Create the `audit_events` table **exactly as defined in `contracts/data-model.md` → Later-phase tables** (do not redefine it here). Write events for: login, invite, project create, grounding launch, simulation launch, verdict view/export, cap change. `GET /workspaces/{id}/audit` (owner only, paged).
**Done when:** migration clean; performing each action writes the matching row; endpoint serves it.

**Files:**
- `services/core-api/alembic/versions/0003_audit_events.py` — creates `audit_events` verbatim from data-model.md (including `idx_audit_workspace`).
- `app/services/audit.py` — single write path for events; no router writes the table directly.
- `app/routers/workspaces.py` — workspace-cap validation on project create/update + `GET /workspaces/{id}/audit` (owner only, `offset`/`limit` per api-surface pagination rule).

**Key functions:**
- `async def record_event(db, *, actor_user_id, workspace_id, action, target_type=None, target_id=None, metadata=None) -> None` — `action` restricted to the enum comment in the DDL (`login`, `invite`, `project_create`, `grounding_launch`, `simulation_launch`, `verdict_view`, `export`, `cap_change`).
- `def validate_workspace_cap(db, workspace_id, new_project_cap: Decimal) -> None` — `SUM(projects.spend_cap_usd) + new_project_cap ≤ workspaces.spend_cap_usd`, else raise `BUDGET_EXCEEDED` (402) — matches mt-03 step 11.

**Tests** (`tests/integration/test_audit.py`): `test_each_action_writes_matching_event` (parametrized over all 8 actions), `test_audit_endpoint_owner_only_403_for_viewer`, `test_audit_pagination`, `test_workspace_cap_sum_exceeded_402`, `test_cap_lowered_frees_headroom`.

**Edge cases:** first login writes both the auto-provision and a `login` event; audit write failure is logged and must not block the user action (but mt-03 step 12 must show all 5 rows — best-effort is not an excuse for loss in the happy path); `actor_user_id` nullable per DDL for pre-auth events; paging must order by `created_at DESC` to match the index.

**Cheap-LLM failure modes:** none — no LLM in this path.

## Stop conditions
- Logto self-host version behaves differently than the checklist assumes → stop, report exact mismatch.
- LiteLLM virtual-key budget API changed → stop, link the upstream docs, ask.
- Never weaken a role check to make a test pass. Report instead.

## Manual test
`../manual-tests/mt-03-auth-billing.md` — all green = MVP complete per PBR §12.
