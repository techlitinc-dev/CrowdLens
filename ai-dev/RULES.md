# Rules for the Coding Agent (paste after CONTEXT.md, every session)

- **R1 Contracts are law.** `ai-dev/contracts/*.md` define every endpoint, table, payload. Never invent or rename one. Contract looks wrong → stop, report.
- **R2 Adapter isolation (AGPL).** Never import fork code. HTTP only, via adapter modules. Only new file allowed inside a fork: `Dockerfile.adapter`.
- **R3 LiteLLM only.** No direct provider SDK calls. Aliases `swarm-model`/`report-model`/`embed-model`, project virtual key on every call.
- **R4 No invented numbers.** Missing value → `null` + reason.
- **R5 No PII.** Strip usernames/avatars/profile links at the adapter boundary.
- **R6 Small diffs.** Only the task. No drive-by refactors. Problem elsewhere → report, don't fix.
- **R7 Prove it.** Task done = done-criteria commands pass; paste real output.
- **R8 Dependencies.** Only ones listed in the task file. Pin versions.
- **R9 Stop conditions (ask, don't guess):** contract/task disagree · fork behaves differently than the adapter contract · missing secret/key · same test fails twice · about to delete/overwrite anything not created this session.
- **R10 Honest errors.** Surface errors verbatim in API + logs. No swallowed exceptions, no fake success.
- **R11 Layout (fixed):**
```
crowdlens/
├── services/core-api/     # FastAPI
├── services/adapters/     # bettafish + miroshark HTTP adapters
├── forks/bettafish/  forks/miroshark/   # AGPL, untouched
├── frontend/
├── infra/                 # docker-compose.yml, litellm/config.yaml
└── tests/                 # integration/, fixtures/
```
