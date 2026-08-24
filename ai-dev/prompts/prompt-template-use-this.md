# Prompt Template — USE THIS

Copy a block below, fill `<...>`, paste into a fresh session. Never reorder the Read lines (cache depends on it). One task per session — always.

---

## 1. Build task (default — use 90% of the time)

```
Read ai-dev/CONTEXT.md and ai-dev/RULES.md, then implement ONLY task <TASK-ID>
in ai-dev/phases/<PHASE-FILE>.

Rules that bite: contracts are law (R1), forks via adapter only (R2), LiteLLM
only (R3), no PII (R5), small diffs (R6), listed dependencies only (R8).

When done: run every command in the task's "Done when" section and paste the
real output. On any stop condition (R9), stop and explain — do not improvise.
```

Fill-ins: `<TASK-ID>` e.g. `P1-T2` · `<PHASE-FILE>` e.g. `phase-1-collectors.md`

---

## 2. Crown-jewel review (after P2-T3 / P2-T6 — run on the strong model)

```
Read ai-dev/contracts/adapter-contract.md §2–3, then review
services/core-api/app/services/<FILE> line by line. No edits. List:
(1) anywhere a number or citation could be fabricated,
(2) every deviation from the contract,
(3) every unhandled error path.
```

Fill-ins: `<FILE>` = `handoff.py` or `verdict.py`

---

## 3. Failure recovery (a manual test or done-check failed)

```
Read ai-dev/RULES.md (R9). A check failed.
Test: <MT-XX STEP N or task done-check>. Expected: <...>. Actual:
<paste full log/output verbatim>

State the root cause in ONE sentence first. Only then the minimal fix.
No unrelated edits.
```

---

## 4. Continue after context loss

```
Read ai-dev/CONTEXT.md and ai-dev/RULES.md. Task <TASK-ID> from
ai-dev/phases/<PHASE-FILE> is partially done. Inspect the repo state,
then finish ONLY that task. Paste the "Done when" outputs.
```

---

## Usage rules (for you, the human)

1. Fresh session per prompt. Don't continue a meandering session — restart with template 4.
2. Never paste file contents into the prompt; reference paths.
3. If output quality drops mid-phase, don't edit CONTEXT.md/RULES.md — that breaks the cache for every later session.
4. One fix attempt per session. Second failure → new session with template 3 + the new log.
5. After a green task: `git add -A && git commit -m "<TASK-ID>: <what>"`.
