# Manual Tests — how to run and record

These checklists are executed **by a human**, not the LLM. Cheap models are convincing when wrong; these tests are the ground truth.

## Rules

1. Run the checklist for a phase **only after the LLM's done-criteria passed**. Green commands ≠ working product.
2. Execute steps **exactly as written**, in order. If a step is impossible, that's a FAIL — don't work around it.
3. Record results directly in a copy of the file (or a results doc):

```
| Step | Result | Evidence |
| 2    | PASS   | pasted output / screenshot path |
| 4    | FAIL   | curl returned 500: <log excerpt> |
```

4. Any FAIL → open a **failure recovery session** (prompts/session-prompts.md) with the failure log. Re-run the entire checklist after the fix — regressions hide in earlier steps.
5. A phase is complete only when every step is PASS. Do not start the next phase on a partial pass.

## Timeboxes

| Checklist | Budget |
|---|---|
| mt-00 backbone | ~30 min |
| mt-01 collectors | ~45 min (includes live API collection) |
| mt-02 first loop | ~60–90 min (a real simulation runs) |
| mt-03 auth/billing | ~45 min |

## The three paranoia checks (repeat every phase)

- **Citations:** pick any claim in any output, click through — does a real source exist?
- **PII:** `SELECT text FROM collected_items` — any usernames/avatars anywhere?
- **Money:** check LiteLLM spend after every LLM-heavy step — is cost tracking real?
