# Session Prompts — copy-paste, one per LLM session

**Rule:** one task per session. Every prompt below is self-contained. Replace `<...>` placeholders. If the agent's context window fills mid-task, open a fresh session with the same prompt plus "continue from where the repo currently is."

## Universal preamble (already embedded in each prompt below)

Every prompt = CONTEXT.md + RULES.md + one task + relevant contract. Do not trim them to save tokens — that is exactly how cheap models drift.

---

## Template

```
You are implementing one task of the CrowdLens build. Read these files in the repo first:
- ai-dev/CONTEXT.md
- ai-dev/RULES.md
- ai-dev/contracts/<contract files the task references>
Then implement ONLY this task: ai-dev/phases/phase-N-....md → Task PN-TM.

Constraints recap: contracts are law; adapters only for fork communication; all LLM
calls via LiteLLM; no PII; small diffs; allowed dependencies only.

When done: run every command in the task's "Done when" section and paste the real
output. If a stop condition triggers, STOP and explain instead of improvising.
```

## Concrete first sessions

**Session 1 — P0-T1 scaffold**
```
Read ai-dev/CONTEXT.md, ai-dev/RULES.md, then implement ONLY task P0-T1 in
ai-dev/phases/phase-0-backbone.md. Paste the output of each "Done when" check.
```

**Session 2 — P0-T2 compose**
```
Read ai-dev/CONTEXT.md, ai-dev/RULES.md, then implement ONLY task P0-T2 in
ai-dev/phases/phase-0-backbone.md. My environment: Docker <version>, ports
4000/5432/7474/7687/9000/9001 free. Paste the output of each "Done when" check.
```

**Session 3 — P0-T3 LiteLLM aliases**
```
Read ai-dev/CONTEXT.md, ai-dev/RULES.md, then implement ONLY task P0-T3 in
ai-dev/phases/phase-0-backbone.md. My DeepSeek key is in .env as DEEPSEEK_API_KEY.
Paste the curl response from the "Done when" section.
```

**…continue task by task, in phase order: P0-T4, P0-T5, then phase 1 tasks, etc.**

## Special sessions

**Human-review gate (after P2-T3 and P2-T6)**
```
You are reviewing, not writing. Read ai-dev/contracts/adapter-contract.md §2 and §3,
then review services/core-api/app/services/handoff.py (or verdict.py) line by line.
List: (1) every place a number or citation could be fabricated, (2) every deviation
from the contract, (3) every unhandled error path. Do not modify code.
```
> If you have access to one stronger model, spend it on this session, not on generation.

**Failure recovery session**
```
A manual test failed. Test: <mt-XX step N>. Expected: <...>. Actual: <paste output/log>.
Read ai-dev/RULES.md R9. Find the root cause FIRST and state it in one sentence.
Only then propose the minimal fix. Do not touch unrelated files.
```

**Red-team session (run once after phase 2)**
```
Attack this build using only ai-dev/RULES.md as the target list: try to make it
(1) run an LLM call outside LiteLLM, (2) store a username, (3) issue a verdict
without convergence, (4) exceed a spend cap. For each: show the exact request/path
or confirm it's impossible. Do not fix anything — report only.
```
