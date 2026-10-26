# AI Knowledge Assistant

A production-style Retrieval-Augmented Generation document assistant. Users upload PDF/DOCX documents, the system ingests them into a private vector index, and users ask grounded questions with traceable citations.

**Status:** Agent team and delivery process are set up. Application implementation begins with Phase 0 (foundation and contracts).

## Source of truth

- [Software Requirements Specification](docs/SRS.md)
- [Agent operating model](docs/agents/operating-model.md)
- [Current delivery status](docs/agents/status.md)
- [Project instructions](AGENTS.md)

## How this project is built

The project uses an OpenCode multi-agent workflow:

- **`supervisor`** is the only agent the user talks to. It plans, delegates, verifies, and owns the final delivery.
- Ten subagents own the modules: `platform`, `documents`, `ingestion`, `retrieval`, `chat`, `frontend`, `infra`, `evaluation`, `qa`, and `reviewer`.
- The supervisor does not edit application code. Every code change is delegated to the owning subagent.
- QA and reviewer gate each phase before the supervisor commits.

## Start the supervisor

Start a new OpenCode session in this directory. The default agent is `supervisor` (otherwise select it from the agent switcher), then send:

```text
Start Phase 0. Read docs/SRS.md and docs/agents/operating-model.md first.
Confirm your architecture and contract plan, then delegate the foundation
work to the platform subagent. Keep me updated at each gate.
```

The supervisor will maintain progress in `docs/agents/status.md`.

## Planned stack

- Backend: Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2 async, Alembic
- Data: PostgreSQL 16, Qdrant, Redis + arq, MinIO
- Auth: Argon2id, JWT access tokens, HttpOnly refresh cookie
- LLM: OpenAI-compatible provider; local `fastembed` embeddings
- Frontend: Next.js, TypeScript, Tailwind CSS, TanStack Query
- Infra: Docker Compose, GitHub Actions

## Notes

- Do not commit `.env` or secrets. Use `.env.example` as the template.
- V1 scope is FR-01–10 and FR-15–17 plus acceptance and evaluation. Summarization, comparison, and tool workflows are later iterations.
- The application is not implemented yet; this repository currently contains the agent team, SRS, and delivery process.
