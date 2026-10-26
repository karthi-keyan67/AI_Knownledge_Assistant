# Delivery Status

## Current phase

**Phase 0 — Foundation and contracts** (not started)

## Confirmed decisions

- V1 scope: FR-01–10, FR-15–17, evaluation, and acceptance criteria. Summarization/comparison/tool workflow are stretch work.
- Stack: FastAPI, PostgreSQL, Qdrant, Redis + arq, MinIO, Next.js.
- LLM: hosted OpenAI-compatible provider; local `fastembed` embeddings.
- Supervisor is orchestrator-only: documentation edits only; all code changes are delegated.
- Built-in Build/Plan agents remain available as escape hatches.

## Task board

| Phase | Task | Owner | Status | Evidence |
|---|---|---|---|---|
| 0 | Architecture and contracts | supervisor + platform | Pending | — |
| 0 | Foundation, auth, DB, interfaces, test harness | platform | Pending | — |
| 1 | Documents and deletion | documents | Pending | — |
| 1 | Ingestion pipeline and worker | ingestion | Pending | — |
| 1 | Compose and CI | infra | Pending | — |
| 2 | Retrieval | retrieval | Pending | — |
| 2 | Chat, citations, conversations | chat | Pending | — |
| 3 | Frontend | frontend | Pending | — |
| 4 | Evaluation | evaluation | Pending | — |
| 5 | Security/acceptance/e2e tests | qa | Pending | — |
| 5 | Independent review | reviewer | Pending | — |
| 5 | README and final report | supervisor | Pending | — |

## Gate log

| Gate | Status | Evidence | Date |
|---|---|---|---|
| 0 | Not started | — | — |
| 1 | Not started | — | — |
| 2 | Not started | — | — |
| 3 | Not started | — | — |
| 4 | Not started | — | — |
| 5 | Not started | — | — |

## Open decisions

- Populate `.env` with the real LLM API key/base URL/model before Phase 2 provider smoke tests and Phase 4 evaluation. Do not paste secrets into chat.

## Next action

Supervisor: read the SRS and operating model, confirm the Phase 0 contract plan, write the architecture/contract docs, and delegate foundation work to `platform`.
