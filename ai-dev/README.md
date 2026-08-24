# CrowdLens AI Development Kit

This folder is the **complete instruction set for building CrowdLens with a cheap LLM coding agent**. It is written so the agent never has to make a design decision — every interface, path, and acceptance check is fixed in advance.

## Folder map

```
ai-dev/
├── README.md                  ← you are here (how to run the process)
├── tutorial.md                ← running this kit with DeepSeek-Flash + Reasonix
├── CONTEXT.md                 ← product context — paste into EVERY session
├── RULES.md                   ← non-negotiable rules — paste into EVERY session
├── contracts/                 ← fixed interfaces — the agent must not deviate
│   ├── adapter-contract.md    ← Core API ↔ BettaFish ↔ MiroShark schemas
│   ├── data-model.md          ← PostgreSQL schema (all tables)
│   ├── api-surface.md         ← Core API REST endpoints v1
│   ├── websocket-protocol.md  ← live simulation feed events (phase 4+)
│   ├── report-blocks.md       ← Report Studio block schema + publish gate
│   ├── platform-services.md   ← Temporal · Lago · Novu · webhooks
│   └── frontend-spec.md       ← design tokens, UI rules, screen inventory
├── phases/                    ← build order — do them in sequence
│   ├── phase-0-backbone.md    ← repo scaffold + OSS services (docker-compose)
│   ├── phase-1-collectors.md  ← Reddit + YouTube collectors → grounding DB
│   ├── phase-2-first-loop.md  ← handoff → simulation → verdict → minimal UI
│   ├── phase-3-auth-billing.md← Logto auth, workspaces, spend caps, metering
│   ├── phase-4-theater-graph.md← simulation theater, persona editor, KG explorer
│   ├── phase-5-report-studio.md← living reports, share links, exports, ask-RAG
│   ├── phase-6-india-multilingual.md← Telegram, app reviews, HI/MR pipeline
│   ├── phase-7-monitoring-bi.md← schedules, shift alerts, dashboards, public API
│   └── phase-8-production-hardening.md← Temporal, prod deploy, DR, security, SSO, on-prem
├── prompts/
│   └── session-prompts.md     ← copy-paste prompts, one per task
└── manual-tests/              ← human verification checklists per phase
    ├── README.md              ← how to run and record manual tests
    ├── mt-00-backbone.md
    ├── mt-01-collectors.md
    ├── mt-02-first-loop.md
    ├── mt-03-auth-billing.md
    ├── mt-04-theater-graph.md
    ├── mt-05-report-studio.md
    ├── mt-06-india.md
    ├── mt-07-monitoring-bi.md
    └── mt-08-production.md    ← launch gate
```

## How to work with a cheap LLM (read this first)

Cheap models fail by **drifting, improvising, and losing context** — not by lacking knowledge. The process below is designed around those three failure modes.

### The session recipe

1. **One task per session.** Never paste a whole phase file and say "build it." Each phase file is split into tasks (T1, T2, ...). One session = one task.
2. **Every session gets the same three inputs**, in this order:
   - `CONTEXT.md` (what the product is)
   - `RULES.md` (what it must never do)
   - The task block from the phase file + any contract file the task references
   - Ready-made prompts in `prompts/session-prompts.md` already combine these.
3. **Demand the done-criteria output.** Every task ends with explicit checks (commands that must pass, files that must exist). Do not accept "done" without the command output pasted back.
4. **You run the manual test.** After each phase, open the matching `manual-tests/mt-*.md` and execute it yourself, step by step. Mark pass/fail, paste evidence. Failures go back to the LLM as a new session with the failure log attached.
5. **Commit after every green task.** Small diffs are the only thing keeping AI output reviewable. Suggested message format: `P1-T2: reddit collector into staging table`.

### Failure handling

| Symptom | Action |
|---|---|
| Same test fails twice | Stop the session. Start a fresh one with: the task, the failing command output, and "find the root cause before writing code." |
| Agent invents an endpoint/field not in `contracts/` | Reject the diff. Point at the contract file. Contracts are the source of truth — if the contract is wrong, change the contract file first, then the code. |
| Agent wants a new dependency | Only allowed if the task file lists it. Otherwise it must justify in writing and wait for your approval. |
| Context window fills up | Start a fresh session. The task files are self-contained precisely so this is safe. |
| Output quality is bad in the **handoff transformer** or **verdict engine** | These are the crown jewels (PBR §4.3). If your cheap model struggles here, rent 1 hour of a stronger model for review — cheap everywhere else, strong on these two files. |

### Build order is mandatory

Phase 0 → 1 → 2 → ... → 8. Each phase's manual test gates the next phase. Do not parallelize phases — the contracts assume the previous phase exists.

### Feature coverage map (phases ↔ features)

| Phase | Features delivered |
|---|---|
| 0–3 (MVP) | F-01–F-10, AUTH-1..4 |
| 4 | F-10 (full theater), F-12 |
| 5 | F-11, F-19 |
| 6 | F-14, F-15 |
| 7 | F-16, F-17, F-18 (v1), F-20, F-39, F-40, F-41, dashboards (F-42 partial) |
| 8 | F-21, AUTH-5..8 |

**Not covered by any phase yet** (add a phase file + manual test when scheduled — same pattern): F-13 (Ad Testing module), F-22–F-26, F-27–F-38 (several require MiroShark/BettaFish fork extensions — plan those as dedicated fork phases with the AGPL rules in RULES.md R2), F-42 (full KPI builder), F-43–F-46.

### What this kit deliberately leaves out

The features listed as uncovered in the coverage map above (F-13, F-22–F-26, F-27–F-38, F-42 full, F-43–F-46) have no phase files yet. Extending the kit is the same pattern: new phase file + new manual test file + contract update if interfaces change. Get phases 0–8 green first.
