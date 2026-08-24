# Data Model — PostgreSQL 16 (core-api owns this schema)

**Version 0.2 · All tables live in database `crowdlens`. Migrations via Alembic.**

Extensions: migration 0001 must run `CREATE EXTENSION IF NOT EXISTS citext;` (users.email). Phase 5 adds `CREATE EXTENSION IF NOT EXISTS vector;` (pgvector, report RAG). `gen_random_uuid()` is core in PG16 — no pgcrypto needed.

PII rule: no column may contain platform-user PII (RULES.md R5). `collected_items` stores the anonymized form defined in the adapter contract.

```sql
-- ── Tenancy ─────────────────────────────────────────────
CREATE TABLE workspaces (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name          text NOT NULL,
  logto_org_id  text UNIQUE,              -- filled in phase 3
  spend_cap_usd numeric(10,2) NOT NULL DEFAULT 100.00,
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE users (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email         citext UNIQUE NOT NULL,
  display_name  text,
  logto_user_id text UNIQUE,              -- filled in phase 3
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE workspace_members (
  workspace_id  uuid REFERENCES workspaces(id) ON DELETE CASCADE,
  user_id       uuid REFERENCES users(id) ON DELETE CASCADE,
  role          text NOT NULL CHECK (role IN ('owner','analyst','viewer')),
  PRIMARY KEY (workspace_id, user_id)
);

-- ── Projects ────────────────────────────────────────────
CREATE TABLE projects (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id  uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  name          text NOT NULL,
  question      text NOT NULL,            -- the decision under test
  status        text NOT NULL DEFAULT 'draft'
                CHECK (status IN ('draft','grounding','grounded','simulating',
                                  'verdict_ready','no_consensus','failed','halted_budget')),
  spend_cap_usd numeric(10,2) NOT NULL,
  litellm_key   text,                     -- per-project virtual key (phase 3)
  created_by    uuid REFERENCES users(id),
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE seed_documents (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id   uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  version      int  NOT NULL,
  storage_key  text NOT NULL,             -- MinIO object key
  filename     text NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  UNIQUE (project_id, version)
);

-- ── Grounding ───────────────────────────────────────────
CREATE TABLE grounding_jobs (
  id          uuid PRIMARY KEY,           -- = job_id sent to BettaFish
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  request     jsonb NOT NULL,             -- exact POST /collect payload
  status      text NOT NULL DEFAULT 'queued'
              CHECK (status IN ('queued','running','done','failed')),
  progress    jsonb,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE collected_items (
  item_id       text PRIMARY KEY,         -- sha256 from adapter contract
  job_id        uuid NOT NULL REFERENCES grounding_jobs(id) ON DELETE CASCADE,
  source        text NOT NULL,
  url           text NOT NULL,
  published_at  timestamptz,
  text          text NOT NULL,
  language      text,
  metrics       jsonb,
  region        text
);
CREATE INDEX idx_items_job ON collected_items(job_id);
CREATE INDEX idx_items_source ON collected_items(source);

CREATE TABLE baseline_reports (
  id          uuid PRIMARY KEY,           -- = report_id from adapter
  job_id      uuid NOT NULL REFERENCES grounding_jobs(id),
  summary     jsonb NOT NULL,             -- adapter §1 GET /analyze shape
  verified    boolean NOT NULL DEFAULT false,  -- citation check passed
  created_at  timestamptz NOT NULL DEFAULT now()
);

-- ── Personas ────────────────────────────────────────────
CREATE TABLE persona_panels (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  version     int  NOT NULL,
  panel       jsonb NOT NULL,             -- adapter §2 persona_panel shape
  edited_by   uuid REFERENCES users(id),  -- null = auto-generated
  created_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (project_id, version)
);

-- ── Simulation ──────────────────────────────────────────
CREATE TABLE simulations (
  id           uuid PRIMARY KEY,          -- = simulation_id sent to MiroShark
  project_id   uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  panel_id     uuid NOT NULL REFERENCES persona_panels(id),
  request      jsonb NOT NULL,            -- exact handoff payload
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE sim_runs (
  id             uuid PRIMARY KEY,        -- = run_id from MiroShark
  simulation_id  uuid NOT NULL REFERENCES simulations(id) ON DELETE CASCADE,
  variant_id     uuid NOT NULL,
  variant_label  text NOT NULL,
  ensemble_index int  NOT NULL,
  status         text NOT NULL DEFAULT 'running'
                 CHECK (status IN ('running','done','failed','checkpointed')),
  results        jsonb,                   -- adapter §2 results shape
  cost_usd       numeric(10,4) NOT NULL DEFAULT 0,
  created_at     timestamptz NOT NULL DEFAULT now()
);

-- ── Verdicts ────────────────────────────────────────────
CREATE TABLE verdicts (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id        uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  simulation_id     uuid NOT NULL REFERENCES simulations(id),
  outcome           text NOT NULL CHECK (outcome IN ('verdict','no_consensus')),
  agreement_score   numeric(4,3),         -- e.g. 0.833
  ranked_variants   jsonb,                -- [{variant_id, label, direction, support}]
  objections        jsonb,                -- [{theme, quotes:[{text, origin:'real'|'sim', ref}]}]
  risk_flags        jsonb,
  confidence        text CHECK (confidence IN ('low','medium','high')),
  report_markdown   text,
  created_at        timestamptz NOT NULL DEFAULT now()
);

-- ── Cost ledger ─────────────────────────────────────────
CREATE TABLE cost_ledger (
  id          bigserial PRIMARY KEY,
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  component   text NOT NULL,              -- 'grounding' | 'analysis' | 'simulation' | 'verdict' | 'report'
  model_alias text,
  tokens_in   bigint,
  tokens_out  bigint,
  cost_usd    numeric(10,6) NOT NULL,
  recorded_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_ledger_project ON cost_ledger(project_id);
```

## Invariants enforced in code (not just SQL)

1. `projects.status` transitions only: `draft → grounding → grounded → simulating → (verdict_ready | no_consensus | failed)`; any state → `halted_budget`.
2. A simulation may only be created if a `baseline_reports` row with `verified = true` exists for the project's latest grounding job (Principle 1).
3. Before any simulation launches: `SUM(cost_ledger.cost_usd) + estimated_cost ≤ projects.spend_cap_usd`, else status `halted_budget`.
4. `verdicts.confidence` derives from `agreement_score`: ≥0.9 high · ≥0.8 medium · below → no verdict.

## Later-phase tables (same database, created by their phase's migration)

These are fixed here so phase files don't define schema ad hoc.

```sql
-- ── Phase 3: audit ──────────────────────────────────────
CREATE TABLE audit_events (
  id              bigserial PRIMARY KEY,
  actor_user_id   uuid REFERENCES users(id),
  workspace_id    uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  action          text NOT NULL,          -- login | invite | project_create | grounding_launch
                                          -- | simulation_launch | verdict_view | export | cap_change | sso_login
  target_type     text,
  target_id       text,
  metadata        jsonb,
  created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_workspace ON audit_events(workspace_id, created_at DESC);

-- ── Phase 5: reports ────────────────────────────────────
CREATE TABLE reports (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  version     int  NOT NULL,
  title       text NOT NULL,
  status      text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','published','archived')),
  language    text NOT NULL DEFAULT 'en' CHECK (language IN ('en','hi','mr')),
  blocks      jsonb NOT NULL,             -- contracts/report-blocks.md shape
  created_by  uuid REFERENCES users(id),
  created_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (project_id, version, language)
);

CREATE TABLE report_shares (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  report_id    uuid NOT NULL REFERENCES reports(id) ON DELETE CASCADE,
  token_hash   text UNIQUE NOT NULL,      -- sha256 of the share token; token itself never stored
  expires_at   timestamptz NOT NULL,      -- ≤ 30 days from creation
  watermark_seed text NOT NULL,
  created_by   uuid REFERENCES users(id),
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE share_views (
  id          bigserial PRIMARY KEY,
  share_id    uuid NOT NULL REFERENCES report_shares(id) ON DELETE CASCADE,
  viewer_ip   inet,
  viewed_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE report_embeddings (          -- pgvector, phase 5; project-scoped RAG
  id          bigserial PRIMARY KEY,
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  source_kind text NOT NULL,              -- 'block' | 'baseline' | 'verdict' | 'item_excerpt'
  source_id   text NOT NULL,
  content     text NOT NULL,
  embedding   vector(1024)                -- dim must match the configured embed-model
);

-- ── Phase 7: monitoring + BI ────────────────────────────
CREATE TABLE monitoring_schedules (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  cron        text NOT NULL,
  sources     jsonb NOT NULL,
  keywords    jsonb NOT NULL,
  active      boolean NOT NULL DEFAULT true,
  last_run_at timestamptz,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE narratives (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  label       text NOT NULL,
  first_seen  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE narrative_snapshots (
  id            bigserial PRIMARY KEY,
  narrative_id  uuid NOT NULL REFERENCES narratives(id) ON DELETE CASCADE,
  captured_at   timestamptz NOT NULL DEFAULT now(),
  volume        int  NOT NULL,
  sentiment     jsonb,                    -- {positive, neutral, negative}
  momentum      numeric(6,3),
  lifecycle     text CHECK (lifecycle IN ('emerging','peaking','declining')),
  driver_item_ids jsonb                   -- collected_items.item_id[]
);
CREATE INDEX idx_narrative_snaps ON narrative_snapshots(narrative_id, captured_at);

CREATE TABLE outcome_logs (               -- Validation Center / F-18
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  verdict_id  uuid NOT NULL REFERENCES verdicts(id),
  outcome     text NOT NULL CHECK (outcome IN ('hit','partial','miss')),
  notes       text,
  logged_by   uuid REFERENCES users(id),
  logged_at   timestamptz NOT NULL DEFAULT now()
);

-- ── Phase 7: public API + webhooks ──────────────────────
CREATE TABLE service_tokens (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  name         text NOT NULL,
  token_hash   text UNIQUE NOT NULL,      -- 'cpub_...' shown once, hash stored
  scopes       jsonb NOT NULL,            -- ['read:projects','read:verdicts',...]
  revoked_at   timestamptz,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE webhook_endpoints (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  url          text NOT NULL,
  secret_hash  text NOT NULL,
  events       jsonb NOT NULL,            -- ['verdict.issued',...]
  active       boolean NOT NULL DEFAULT true,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
  id           bigserial PRIMARY KEY,
  endpoint_id  uuid NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
  event        text NOT NULL,
  payload      jsonb NOT NULL,
  status       text NOT NULL DEFAULT 'pending'
               CHECK (status IN ('pending','delivered','failed')),
  attempts     int NOT NULL DEFAULT 0,
  created_at   timestamptz NOT NULL DEFAULT now()
);

-- ── Phase 8: enterprise ─────────────────────────────────
CREATE TABLE workspace_branding (         -- white-label exports (phase 5 uses this too)
  workspace_id uuid PRIMARY KEY REFERENCES workspaces(id) ON DELETE CASCADE,
  logo_key     text,                      -- MinIO object key
  accent_color text,                      -- hex
  updated_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE sso_configs (
  workspace_id uuid PRIMARY KEY REFERENCES workspaces(id) ON DELETE CASCADE,
  protocol     text NOT NULL CHECK (protocol IN ('saml','oidc')),
  logto_connector_id text NOT NULL,
  enabled      boolean NOT NULL DEFAULT false
);
```

Later-phase invariants:

5. Share tokens are only ever stored hashed; expiry ≤30 days is enforced at creation, not at read time.
6. `report_embeddings` queries MUST filter `project_id =` first — vector similarity is never cross-project.
7. `outcome_logs` rows are append-only (no UPDATE) — the accuracy page's credibility depends on it.
