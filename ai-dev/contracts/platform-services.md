# Platform Services Contract — Temporal · Lago · Novu · Webhooks

**Version 0.1 · How core-api uses the platform services. Fixed names so agent code is consistent.**

## Temporal (introduced in phase 8 — before that, background tasks suffice)

Task queue: `crowdlens`. Workflows (names fixed):

| Workflow | Input | Signals | Notes |
|---|---|---|---|
| `GroundingWorkflow` | `{job_id}` | `cancel` | polls adapter, upserts items; heartbeat 30s |
| `EnsembleWorkflow` | `{simulation_id}` | `halt_budget` | adaptive ensemble loop (adapter §3); child workflow per run; checkpoints after every round-poll |
| `MonitoringWorkflow` | `{project_id, cron}` | `run_now` | scheduled re-collection + re-simulation |
| `ExportWorkflow` | `{report_id, format}` | — | PDF/PPTX render → MinIO |

Retry policy (all): 5 attempts, exponential 10s→5min. Every workflow resumable after restart — idempotency keys = the entity UUIDs.

## Lago (billing)

Billable metrics (event `code`s, sent via Lago batch API, `external_customer_id = workspace_id`):

| code | when | properties |
|---|---|---|
| `decision_cycle` | verdict or no_consensus issued | `project_id`, `agreement_score` |
| `monitoring_run` | scheduled re-simulation completes | `project_id` |
| `export` | PDF/PPTX delivered | `format` |
| `api_call` | public-API request (metered tiers) | `endpoint` |

Credits: 1 credit = 1 `decision_cycle`. Plan mapping (PBR §14): Starter 10, Agency 60. Core-api checks Lago wallet balance before launching a simulation — insufficient → `402` with top-up link. Lago is the billing source of truth; `cost_ledger` remains the internal cost record (they reconcile nightly, drift >1% → alert).

## Novu (notifications)

Trigger ids (fixed), payload shape `{project_name, workspace_name, ...}`:

| trigger | when | channels |
|---|---|---|
| `verdict_ready` | verdict issued | email, in-app |
| `no_consensus` | ensemble failed to converge | email, in-app |
| `budget_halt` | cap tripped | email, in-app, WhatsApp (opt-in) |
| `sentiment_shift` | monitoring detects shift > threshold | email, WhatsApp (opt-in) |
| `weekly_digest` | Monday 09:00 workspace-local | email |
| `share_viewed` | a share link is opened (agency analytics) | in-app |

Every notification deep-links to the relevant screen. WhatsApp only via Novu's WhatsApp provider, opt-in stored per user.

## Public webhooks (F-20)

Workspace-configured endpoints, signed `X-CrowdLens-Signature: HMAC-SHA256(secret, body)`:

| event | body key fields |
|---|---|
| `grounding.completed` | `project_id`, `job_id`, `item_counts` |
| `simulation.completed` | `project_id`, `simulation_id`, `runs`, `outcome` |
| `verdict.issued` | `project_id`, `verdict_id`, `confidence`, `agreement_score` |
| `budget.halted` | `project_id`, `spent`, `cap` |

Delivery: at-least-once, 5 retries over 1h; workspace can replay from `GET /workspaces/{id}/webhooks/deliveries`.
