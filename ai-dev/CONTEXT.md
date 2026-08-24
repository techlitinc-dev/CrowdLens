# CrowdLens — Context (paste first, every session)

CrowdLens: real cited opinion data in → grounded multi-agent simulation → verdict out. Pipeline order is fixed:

```
seed doc → GROUNDING (real cited data) → persona panel → SIMULATION (variants, ensemble) → VERDICT
```

Full spec: `../readme.md` (PBR) · features: `../features.md` · stories: `../user-stories.md` — read only when a task says so.

## Principles (bind all code)
1. Grounded or nothing — no sim without verified grounding.
2. Evidence-first — every claim traces to a real citation or labeled sim run.
3. Ensemble — verdicts need ≥80% run agreement (adaptive, 3–7 runs); no agreement → `no_consensus`.
4. No invented numbers — `null` + reason, never estimates dressed as data. UI shows uncertainty.
5. Domain-agnostic — one pipeline, all verticals.

## Stack (fixed, no substitutes)
Core API: Python 3.12 + FastAPI · Frontend: React 18 + TS + Vite + Tailwind + shadcn/ui · Real-data: BettaFish fork (Flask, AGPL, adapter-only) · Simulation: MiroShark fork (Flask, AGPL, adapter-only) · LLM: LiteLLM only, aliases `swarm-model` (DeepSeek-V4-Flash) / `report-model` (V4-Pro) / `embed-model` (phase 5+) · DB: PostgreSQL 16 · Graph: Neo4j 5 (MiroShark's) · Cache: Redis 7 · Auth: Logto (phase 3) · Tracing: Langfuse · Storage: MinIO

## Cost config (PBR §11.1, fixed)
120 agents × 18 rounds · early-stop after 3 flat rounds · ensemble 3→7 @ ≥80% · ≤3 variants · caching ON. Sim layer ≤$3/cycle, verdict ≤$25 all-in.

## Compliance floor
No platform-user PII persisted. Store: item id, platform, timestamp, text, public metrics. Never: usernames, avatars, profile links.
