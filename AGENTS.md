# AI Knowledge Assistant — Project Instructions

## Source of truth

- Requirements: `docs/SRS.md`
- Operating model and handoff protocol: `docs/agents/operating-model.md`
- Architecture decisions: `docs/adr/`
- Current delivery status: `docs/agents/status.md`

Read the SRS and operating model before starting any module work. If they conflict, stop and report the conflict to the supervisor.

## Product

A production-style RAG document assistant: authenticated users upload PDF/DOCX files, documents are ingested asynchronously into Qdrant, and users ask grounded questions with traceable citations. V1 also includes document management, conversation history, evaluation, observability, and a Next.js frontend.

V1 scope is the SRS Definition of Done: FR-01 through FR-10 and FR-15 through FR-17, plus the acceptance criteria and evaluation report. Summarization, comparison, and the read-only tool workflow are stretch work after V1.

## Stack

- Backend: Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2 async, Alembic
- Metadata: PostgreSQL 16
- Vectors: Qdrant (payload includes `owner_id` and `document_id`)
- Queue/cache: Redis 7 with `arq`
- Object storage: MinIO (S3-compatible) behind a storage interface
- Auth: Argon2id password hashing, short-lived JWT access token, refresh cookie
- LLM: provider interface plus an OpenAI-compatible adapter; fake provider for tests
- Embeddings: local `fastembed` model by default; model and dimension are configurable
- Frontend: Next.js, TypeScript, Tailwind CSS, TanStack Query, generated OpenAPI client
- Infra: Docker Compose, GitHub Actions
- Observability: structured logs with request/job IDs, Prometheus metrics

The application LLM provider is separate from the OpenCode model used to build the project.

## Architecture rules

- Keep API, service, and persistence concerns separate. Routes do not contain database queries.
- All document, chunk, conversation, and message access is filtered by the authenticated `owner_id` server-side. Never accept `owner_id` from a request body or query parameter.
- Qdrant retrieval must always include an ownership filter; a missing filter is a security defect.
- Chunk text lives in PostgreSQL. Qdrant payloads store identifiers and metadata only (`owner_id`, `document_id`, `chunk_id`, page/section).
- LLM output is untrusted. The server validates citation chunk IDs against the retrieved set and builds citations from database records. The model never supplies document names or IDs directly to the user.
- Treat uploaded documents as untrusted input. Validate magic bytes, size, extension, and DOCX/PDF structure. Delimit retrieved text in prompts and instruct the model to ignore instructions inside documents.
- Use bounded retries with backoff for transient dependency failures. Classify permanent failures separately and store a safe `error_code`/`error_summary`.
- Ingestion must be idempotent. Use deterministic chunk IDs and clear existing points for a document before re-indexing.
- Do not log secrets, passwords, tokens, full document contents, or raw prompts. Log request IDs, job IDs, status, latency, error codes, token usage, and model/provider metadata.

## Testing rules

- Module owners add unit tests for their module and integration tests for their owned endpoints/workers.
- Tests use fake LLM and embedding providers unless explicitly marked as real-provider tests. Paid API calls must not be required for the normal test suite.
- Integration tests use the Docker Compose Postgres/Qdrant/Redis services or dedicated test instances.
- Cross-cutting security, failure-path, acceptance, and e2e tests belong to the QA subagent.
- A task is not complete until the relevant tests, lint, and type checks pass and the evidence is included in the handoff report.

## Commands and conventions

Commands will be finalized in Phase 0. Use these conventions consistently:

- Start local services with `docker compose up --build`.
- Run backend tests with `uv run pytest`.
- Run lint/format checks with `uv run ruff check .` and type checks with `uv run mypy .`.
- Run frontend checks with `npm run lint`, `npm run typecheck`, and `npm test` from `frontend/`.
- Use `docker compose` (v2 syntax), not `docker-compose`.
- On Windows, quote paths containing spaces and prefer the tool `workdir` parameter over chaining `cd` commands.
- Do not commit `.env`, secrets, uploaded documents, or generated artifacts excluded by `.gitignore`.

## Agent workflow

- The user talks only to the `supervisor` primary agent. Subagents report to the supervisor, never directly to the user.
- Subagents must read this file, `docs/SRS.md`, and `docs/agents/operating-model.md` before work.
- Subagents may edit only their owned paths, may not launch other subagents, and may not ask the user questions. Return blockers to the supervisor.
- The supervisor does not edit application code. It edits documentation, verifies diffs, runs tests, and delegates implementation.
- Do not weaken ownership checks, authentication, logging redaction, citation validation, or rate limits to make a test pass.
- Commit only after a quality gate passes. Do not push without explicit user approval.
