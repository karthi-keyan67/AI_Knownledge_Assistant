# Agent Operating Model

This document defines how the AI Knowledge Assistant is delivered by the agent team. It is read by the supervisor and every subagent.

## 1. Roles

| Agent | Mode | Owns |
|---|---|---|
| `supervisor` | primary | User interface, task allocation, contracts, quality gates, documentation, final delivery |
| `platform` | subagent | Core config, auth, database models, schemas, migrations, shared interfaces, vector-store adapter, health, logging, test harness |
| `documents` | subagent | Upload validation, storage, document CRUD/status, job status endpoint, deletion orchestration |
| `ingestion` | subagent | PDF/DOCX parsing, chunking, embeddings, worker, retries, idempotency, Qdrant writes/deletes |
| `retrieval` | subagent | Owner-filtered search, top-k, metadata filters, dense baseline, hybrid retrieval, reranking |
| `chat` | subagent | LLM provider, prompts, grounded answers, citation validation, insufficient evidence, conversations/history |
| `frontend` | subagent | Next.js UI, auth flows, upload/status, chat/citations, loading/error/empty states, accessibility |
| `infra` | subagent | Compose, Dockerfiles, environment example, CI, healthchecks, local startup |
| `evaluation` | subagent | Versioned dataset, runner, metrics, baseline vs hybrid/rerank, reports |
| `qa` | subagent | Cross-cutting integration, security, failure-path, acceptance, and e2e tests |
| `reviewer` | subagent | Independent read-only audit and severity-ranked findings |

Only `supervisor` is a primary agent. Subagents run in child sessions and cannot launch further subagents. The user talks only to the supervisor.

## 2. Ownership and conflict rules

- Each subagent may edit only the paths granted in its agent configuration.
- Shared files are owned by `platform`: models, schemas, migrations, dependency files, router registration, test fixtures, shared interfaces, and the vector-store adapter.
- If a module needs a shared contract change, it reports the need to the supervisor. The supervisor delegates the change to `platform`; modules do not edit shared files.
- One writer at a time per file area. Parallel subagents are allowed only for disjoint paths or read-only research.
- The supervisor rejects any handoff with out-of-scope edits or unverified claims.
- The reviewer never edits files. QA edits only test paths. Evaluation edits only `evaluation/**`.

## 3. Delivery phases and gates

| Phase | Owner(s) | Gate evidence |
|---|---|---|
| 0. Foundation and contracts | platform | Compose starts; auth and health tests pass; schema/OpenAPI review; no secrets |
| 1. Documents and ingestion | documents, ingestion, infra | PDF/DOCX upload reaches COMPLETED; retry/failure paths; deletion removes vectors; cross-user access blocked |
| 2. Retrieval and chat | retrieval, chat | Cited answer; insufficient-evidence response; cross-user isolation; prompt-injection test |
| 3. Frontend | frontend | End-to-end auth/upload/chat flow; citations visible; error and loading states; keyboard access |
| 4. Evaluation | evaluation, retrieval, chat | 50+ question dataset; baseline vs hybrid/rerank; reproducible metrics; manual sample review |
| 5. Hardening and delivery | qa, reviewer, infra, supervisor | SRS acceptance suite, security tests, metrics, README, final report; reviewer sign-off |

Phase 6 (summarization, comparison, read-only tool workflow) begins only after V1 Definition of Done is met and the user approves the stretch scope.

## 4. Handoff brief template

The supervisor must provide a complete brief because every subagent starts with fresh context.

```text
MODULE / TASK:
OBJECTIVE:
READ FIRST:
  - docs/SRS.md (specific sections)
  - docs/agents/operating-model.md
  - AGENTS.md
  - docs/architecture.md and contracts
CONTEXT:
  - What already exists
  - Relevant files and interfaces
OWNED PATHS:
  - Paths you may edit
DO NOT EDIT:
  - Shared or other module paths
CONTRACTS:
  - DB models/schemas
  - API request/response shapes
  - Service interfaces
ACCEPTANCE CRITERIA:
  - Observable behavior required
TESTS REQUIRED:
  - Unit/integration/security cases
  - Exact commands to run
CONSTRAINTS:
  - Security, performance, no secrets, no logging content
REPORT FORMAT:
  - Status: DONE | PARTIAL | BLOCKED
  - Files changed
  - Contracts used or changed
  - Tests run and results
  - Acceptance evidence
  - Risks/known issues
  - Recommended next step
```

The supervisor should continue an existing subagent session when iterating on the same module, and start a fresh session for a new phase or when context is stale.

## 5. Verification protocol

After every handoff the supervisor must:

1. Inspect `git status` and `git diff` for out-of-scope edits.
2. Compare the changes with the brief and acceptance criteria.
3. Run the relevant tests or have QA run them; do not rely on the subagent's claim alone.
4. Check security-sensitive paths: ownership filters, auth, upload validation, citation validation, logging redaction.
5. Ask the reviewer to audit the gate.
6. Delegate fixes to the owning subagent and repeat until the gate passes.
7. Record the evidence in `docs/agents/status.md` and the handoff/review documents.

The supervisor must not mark work complete when tests fail, evidence is missing, or a critical reviewer finding is unresolved.

## 6. Quality gates

A gate passes only when:

- The acceptance criteria for the phase are demonstrated with test output or a reproducible manual check.
- Unit, integration, and relevant security tests pass.
- Lint and type checks pass.
- No out-of-scope edits or secrets are present.
- Ownership and authorization behavior is verified.
- Observability requirements for the phase are present (request/job IDs, latency, token usage, error codes).
- Documentation is updated.
- Reviewer findings are resolved or explicitly recorded as accepted known limitations.

## 7. V1 Definition of Done

- A user can upload a valid PDF or DOCX and see status change to COMPLETED.
- A user can ask a question and receive an answer with at least one traceable citation when evidence exists.
- An unsupported question returns a clear insufficient-evidence response.
- Cross-user document/conversation access is blocked for read, delete, chat, and retrieval.
- Deleting a document removes or invalidates its vectors and access paths.
- Ingestion failures are visible with a safe reason; transient failures respect the retry limit.
- Automated tests cover API, retrieval, ownership, and failure handling.
- Evaluation runs are reproducible against a versioned dataset and compare baseline with at least one improvement.
- A new developer can start the app locally from the README and Docker Compose.
- README includes architecture, setup, environment variables, API examples, known limitations, and measured evaluation results.

## 8. Security and privacy rules

- Never log secrets, passwords, tokens, full document text, raw prompts, or raw retrieved context.
- Never read or expose `.env` contents. Use `.env.example` for documentation.
- Validate uploads as untrusted input: magic bytes, extension/MIME agreement, size, page count, and DOCX/PDF structure.
- Apply server-side ownership checks on every document, job, chunk, conversation, and message operation.
- Qdrant queries must include an `owner_id` filter even when a document ID is supplied.
- Server-validate citations. Do not trust model-provided source names, document IDs, or citation IDs.
- Treat document text as untrusted instructions; delimit it in prompts and test prompt injection with a canary.
- Use bounded retries and timeouts for LLM, embedding, storage, and vector operations.
- Do not weaken security controls to make tests pass.

## 9. Status and audit documents

The supervisor maintains:

- `docs/agents/status.md` — current phase, tasks, owners, status, gate evidence, open decisions.
- `docs/agents/sessions.md` — subagent session IDs for continued work.
- `docs/agents/handoffs/` — the brief and report for each task.
- `docs/agents/reviews/` — reviewer findings and resolution status.
- `docs/adr/` — architecture decision records.
- `docs/architecture.md` — final architecture and data flow.
- `docs/final-report.md` — final evidence, evaluation summary, limitations, demo steps.

## 10. Git and user communication

- Commit after each passing gate with a clear message. Do not push without explicit user approval.
- Keep the user informed with concise phase updates, decisions needed, risks, and evidence. Do not dump raw logs.
- Ask the user only for decisions, missing access/credentials, or scope changes. Tell them to set credentials in `.env`; never ask them to paste secrets into chat.
- If a subagent fails twice on the same issue, diagnose the cause, reassign or narrow the task, and escalate to the user if the blocker is external.
