# CrowdLens — Condensed Product Context

> Paste this file into every LLM session, before anything else.

## What we are building

CrowdLens tells a business **what the world thinks now** (real, cited data) and simulates **what the world will think next** (grounded multi-agent simulation) — so decisions are tested before money is spent.

The pipeline, always in this order:

```
seed document → GROUNDING (real data, cited) → persona panel
  → SIMULATION (counterfactual variants, ensemble of runs)
  → VERDICT (ranked options, objections with quotes, confidence)
```

Full spec: `../CrowdLens_PBR.md` · features: `../features.md` · stories: `../user-stories.md`

## Product principles (bind every line of code)

1. **Grounded or nothing** — no simulation runs without a real-data grounding layer.
2. **Evidence-first** — every claim must be traceable to a real citation or a labeled simulation run.
3. **Ensemble by default** — a single run is an anecdote; verdicts require ≥80% agreement across runs (adaptive ensemble, 3–7 runs).
4. **Confidence, not theater** — never invent precise numbers (no fake CTRs, vote shares, revenue). Report direction, themes, and uncertainty.
5. **Domain-agnostic** — one pipeline serves all verticals; advertising is just one module.

## Tech stack (fixed — do not substitute)

| Layer | Choice |
|---|---|
| Core API | Python 3.12 + FastAPI |
| Frontend | React 18 + TypeScript + Vite + Tailwind + shadcn/ui |
| Real-data engine | BettaFish fork (Flask service, AGPL — internal only, behind adapter) |
| Simulation engine | MiroShark fork (Flask service, AGPL — internal only, behind adapter) |
| LLM gateway | LiteLLM — ALL LLM calls go through it, no exceptions |
| LLM models | alias `swarm-model` = DeepSeek-V4-Flash · alias `report-model` = DeepSeek-V4-Pro · alias `embed-model` = embedding model for report RAG (phase 5+; any LiteLLM-supported provider) |
| Core DB | PostgreSQL 16 |
| Graph DB | Neo4j 5 (used by MiroShark) |
| Cache/queue | Redis 7 |
| Auth | Logto (phase 3) |
| Tracing/evals | Langfuse |
| Object storage | MinIO (S3-compatible) |

## Cost config (PBR §11.1 — fixed)

120 agents × 18 rounds per run · early-stop after 3 flat rounds · adaptive ensemble (start 3, ≥80% agreement, cap 7) · ≤3 variants at MVP · prompt caching ON. Target: ≤$3 simulation-layer cost per decision cycle, ≤$25 all-in per verdict.

## Compliance floor

No PII of platform users is ever persisted. Store: post id, platform, timestamp, text, public metrics. Never store: real usernames, avatars, profile links. Aggregate-only publishing.
