# Tutorial — Building CrowdLens with DeepSeek-V4-Flash + Reasonix

How to execute the `ai-dev/` kit using [Reasonix](https://github.com/esengine/deepseek-reasonix) — a terminal coding agent built specifically around DeepSeek's prefix cache — so the MVP build costs single-digit dollars in model tokens.

**Why this pairing:** DeepSeek bills cached input at **$0.0028/M vs $0.14/M uncached** (50× cheaper) and output at $0.28/M. Reasonix's whole design is keeping your prompt prefix *byte-stable* so DeepSeek's automatic cache keeps hitting across a long build session ([field notes](https://wuu73.org/aiguide/infoblogs/coding_agents/reasonix.html), [overview](https://www.oflight.co.jp/en/columns/deepseek-reasonix-terminal-coding-agent-2026)). The `ai-dev/` kit was already written cache-first: CONTEXT.md + RULES.md + contracts are meant to be identical at the front of every session — which is exactly the prefix Reasonix preserves.

---

## 1. Install & setup (~10 min)

Prereqs: Node.js ≥ 22 (or Go), a DeepSeek API key from [platform.deepseek.com](https://platform.deepseek.com).

```bash
npm i -g reasonix        # single binary; Go install also available — see repo README
reasonix setup           # interactive: paste DeepSeek API key, pick default model
reasonix                 # start a session in the repo root
```

Setup choices that matter:

- **Default model: `deepseek-v4-flash`** (non-thinking). Do NOT use the legacy names `deepseek-chat`/`deepseek-reasoner` — they were [retired 2026-07-24](https://api-docs.deepseek.com/updates/) and alias to Flash anyway.
- **Enable thinking mode only per-session** (it maps to the same model's reasoning pass) for the crown-jewel reviews in §5. Day-to-day building stays non-thinking — thinking tokens are output tokens, i.e. the expensive kind.
- If Reasonix asks for a provider/endpoint: DeepSeek's OpenAI-compatible endpoint `https://api.deepseek.com` is the default; leave it.

## 2. The one rule that saves the money

DeepSeek's prefix cache matches **from the first byte**. Anything that changes near the *front* of your prompt invalidates everything after it. So:

**Immutable at the front, variable at the back — always in this order:**

```
[1] ai-dev/CONTEXT.md      ← never edited during a phase
[2] ai-dev/RULES.md        ← never edited during a phase
[3] ai-dev/contracts/*.md  ← edited rarely, and only between sessions
[4] the task block          ← the only part that changes per session
```

Concrete habits:

- **Don't touch CONTEXT.md / RULES.md mid-phase.** Even a one-word edit resets the cache for every following session. Batch doc fixes between phases.
- **Use the prompts in `ai-dev/prompts/session-prompts.md` verbatim.** They're already ordered stable-first.
- **Reference files by path** ("Read ai-dev/contracts/adapter-contract.md §2") instead of pasting file contents — Reasonix reads them via tools, and file contents don't sit in your prompt prefix.
- **One task per session** (per ai-dev/README). Long meandering sessions accumulate rewritten history — the cache killer. A fresh session with the same stable prefix re-hits the cache immediately.
- After P0-T1, the repo has `AGENTS.md` (CONTEXT+RULES inlined) — Reasonix picks it up automatically each session, so your typed prompt shrinks to just the task.

## 3. The working loop (per task)

```bash
cd crowdlens                # repo root (created in P0-T1; before that, the dir holding ai-dev/)
reasonix                    # new session
```

Then, from `ai-dev/prompts/session-prompts.md`, paste the session prompt for the current task, e.g.:

```
Read ai-dev/CONTEXT.md, ai-dev/RULES.md, then implement ONLY task P1-T2 in
ai-dev/phases/phase-1-collectors.md. Paste the output of each "Done when" check.
```

1. Let it work. Reasonix shows live token/cost usage — glance at the **cache-hit rate**; ≥80% after the first minutes is normal with a stable prefix, and each hit is billed at 1/50th.
2. When it pastes done-criteria output: **verify the commands yourself once** (cheap models are convincing when wrong).
3. Commit: `git add -A && git commit -m "P1-T2: bettafish adapter module"`.
4. Close the session. Next task = new session, same stable prefix → cache hit from the first request.
5. After each phase: you run the matching `ai-dev/manual-tests/mt-*.md` checklist. Failures go back via the failure-recovery prompt.

## 4. Token budget — what the MVP build should cost

Rough, honest numbers for phases 0–3 (~20 task sessions):

| Component | Estimate | Cost @ Flash |
|---|---|---|
| Stable prefix (CONTEXT+RULES+AGENTS, ~6K tokens — slimmed for exactly this — × 20 sessions × cache reads) | ~120K cached input | ~$0.01 |
| Repo context reads, tool output, history (the real input bulk, mostly cache-hitting) | ~15M input, ~80% cached | ~$0.45 |
| Generated code + tests (output) | ~3M output | ~$0.84 |
| 2 crown-jewel reviews on `deepseek-v4-pro` (thinking) | ~0.5M in / 0.3M out | ~$0.45 |
| **Total MVP build** | | **~$2–5** |

If your actual bill lands at 5–10× this, the cache is being defeated — see §6.

## 5. When to spend more on purpose

Two sessions deserve the strong model (`deepseek-v4-pro`, thinking mode): the reviews of `handoff.py` (P2-T3) and `verdict.py` (P2-T6) — the copy-paste review prompts are in `session-prompts.md`. Everything else — scaffold, collectors, adapters, UI, auth — Flash handles fine *because the contracts remove design freedom*. If Flash flails on any task (same test failing twice), follow the failure table in `ai-dev/README.md` instead of brute-forcing retries.

## 6. Troubleshooting the savings

| Symptom | Likely cause | Fix |
|---|---|---|
| Cache-hit rate <50% | Stable files edited mid-phase, or history rewriting in a long session | Revert doc edits; start fresh sessions per task |
| Bill >> $5 | Thinking mode left on, or retries of the same failing task | Thinking off for routine tasks; stop-condition → fresh session with the failure log |
| Agent ignores a contract | Prefix too long / contract not referenced | Point at the contract file + section explicitly in the prompt |
| Model name errors (`invalid_request_error`) | Legacy alias used | Use exactly `deepseek-v4-flash` / `deepseek-v4-pro` |

## 7. What stays human

Reasonix + Flash writes the code; **you** run the manual tests. The three paranoia checks from `ai-dev/manual-tests/README.md` — citations, PII, money — are non-delegable. The whole point of this stack is: cheap tokens for typing, human judgment for trust.
