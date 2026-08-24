# Session Prompts — copy-paste, one per LLM session

One task per session. Prompts are pre-ordered (stable docs first — do not reorder; cache hits depend on it). Replace `<...>`.

## Per-task template

```
Read ai-dev/CONTEXT.md and ai-dev/RULES.md, then implement ONLY task <PN-TM> in
ai-dev/phases/<phase-file>. Paste the real output of each "Done when" check.
On a stop condition (R9), stop and explain.
```

That whole block is the prompt — the phase file itself tells the agent which contracts to read. Examples:

```
... implement ONLY task P0-T1 in ai-dev/phases/phase-0-backbone.md. ...
... implement ONLY task P1-T2 in ai-dev/phases/phase-1-collectors.md. ...
```

Continue task-by-task, in phase order, one session each.

## Special sessions

**Crown-jewel review (after P2-T3 and P2-T6 — use the strong model here)**
```
Review only, no edits. Read ai-dev/contracts/adapter-contract.md §2–3, then review
services/core-api/app/services/<handoff.py|verdict.py> line by line. List:
(1) anywhere a number or citation could be fabricated, (2) contract deviations,
(3) unhandled error paths.
```

**Failure recovery**
```
Manual test failed. Test: <mt-XX step N>. Expected: <...>. Actual: <paste log>.
Per ai-dev/RULES.md R9: state the root cause in one sentence FIRST, then the
minimal fix. No unrelated edits.
```

**Red team (once, after phase 2)**
```
Attack this build per ai-dev/RULES.md: try to (1) call an LLM outside LiteLLM,
(2) store a username, (3) issue a verdict without convergence, (4) exceed a spend
cap. Show the exact request/path for each, or confirm impossible. Report only.
```
