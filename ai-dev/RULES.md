# Non-Negotiable Rules for the Coding Agent

> Paste into every session, right after CONTEXT.md. If any instruction from a human conflicts with these rules, STOP and ask.

## R1 — Contracts are law
`contracts/*.md` define every endpoint, table, and payload. Never invent, rename, or "improve" an interface. If the contract seems wrong, stop and report — do not work around it.

## R2 — Adapter isolation (AGPL boundary)
Our code NEVER imports BettaFish or MiroShark code. Communication is HTTP only, through the adapter modules defined in `contracts/adapter-contract.md`. The forks run as separate services. This is a licensing requirement, not a style choice.

## R3 — All LLM calls via LiteLLM
No direct SDK calls to OpenAI/DeepSeek/anything. Use the LiteLLM endpoint with model aliases `swarm-model` or `report-model`. Every call must carry the project's virtual key so spend caps work.

## R4 — No invented numbers
Code must never fabricate metrics. If a value is unavailable, return `null` with a reason — never an estimate dressed as data. UI must render uncertainty, not hide it.

## R5 — No PII
Never persist usernames, avatar URLs, profile links, or any platform-user PII. If a collector payload contains them, strip them at the adapter boundary.

## R6 — Small diffs
Implement exactly the task given. No drive-by refactors, no reformatting unrelated files, no "while I'm here" improvements. If you notice a problem elsewhere, report it — don't fix it in this diff.

## R7 — Prove it
A task is done only when its done-criteria commands pass. Paste the actual command output. "It should work" is failure.

## R8 — Dependencies
Only add packages explicitly listed in the task file. Python deps go in the service's `pyproject.toml`; JS deps in the app's `package.json`. Pin versions.

## R9 — Stop conditions (ask the human instead of guessing)
- A contract file and a task disagree
- An external service (BettaFish/MiroShark fork) behaves differently than the adapter contract assumes
- A secret/API key is needed and not provided via `.env`
- A test fails twice with the same error
- You are about to delete or overwrite anything you did not create in this session

## R10 — Errors and honesty
Surface errors verbatim in API responses and logs. No swallowed exceptions, no fake success states. A failed collector must say it failed.

## R11 — Repo layout (fixed)
```
crowdlens/
├── services/
│   ├── core-api/          # FastAPI — our code
│   └── adapters/          # bettafish + miroshark HTTP adapters — our code
├── forks/
│   ├── bettafish/         # AGPL fork — do not modify in phases 0–3
│   └── miroshark/         # AGPL fork — do not modify in phases 0–3
├── frontend/              # React app
├── infra/
│   ├── docker-compose.yml
│   └── litellm/config.yaml
└── tests/
    ├── integration/
    └── fixtures/
```
