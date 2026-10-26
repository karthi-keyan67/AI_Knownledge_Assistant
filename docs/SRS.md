# Software Requirements Specification

## AI Knowledge Assistant

Portfolio Project · Version 1.0 · October 2026

**Purpose:** Define the functional, technical, security, and quality requirements for a production-style document question-answering application using Retrieval-Augmented Generation (RAG). This document is intended to guide implementation, testing, and portfolio presentation.

## 1. Project Overview

The AI Knowledge Assistant enables authenticated users to upload documents, organize a private knowledge base, ask natural-language questions, and receive answers grounded in retrieved document passages with traceable source citations. The application also provides document comparison and summarization, background ingestion, evaluation, and operational monitoring.

### 1.1 Goals

- Build an end-to-end AI application rather than a standalone chatbot demo.
- Provide answers grounded in authorized documents and clearly indicate when evidence is insufficient.
- Measure retrieval and answer quality using a repeatable evaluation dataset.
- Demonstrate reliable backend engineering: testing, retries, logging, security, and deployment.
- Produce a portfolio artifact that can be explained in technical interviews.

### 1.2 Out of Scope for Version 1

- Training a foundation model from scratch.
- Fine-tuning a model.
- Autonomous agents that can take irreversible external actions.
- Enterprise SSO, billing, and multi-region high availability.
- Guaranteeing that LLM answers are always correct.

## 2. Users and Roles

| Role | Permissions | Restrictions |
|---|---|---|
| Registered user | Upload, list, view, delete own documents; ask questions; view own chat history and jobs. | Cannot access another user's documents or conversations. |
| Developer/operator | Configure deployment, inspect aggregate operational logs, manage service configuration. | Must not expose document contents or secrets through logs. |

## 3. Functional Requirements

- **FR-01 User authentication:** The system shall allow a user to register or sign in securely. Protected APIs shall require a valid access token.
- **FR-02 Document upload:** The system shall accept PDF and DOCX files initially, validate file type and size, assign a unique document ID, and associate the document with its owner.
- **FR-03 Document ingestion:** The system shall extract text, preserve available page/section metadata, split text into configurable chunks, create embeddings, and store chunks in Qdrant.
- **FR-04 Background processing:** Document ingestion shall run asynchronously. The API shall return a job/document status such as QUEUED, PROCESSING, COMPLETED, or FAILED.
- **FR-05 Ingestion retry:** Transient ingestion failures shall be retried with a bounded retry policy. Permanent failures shall be recorded with a safe, user-readable reason.
- **FR-06 Document management:** Users shall be able to list their documents, view status/metadata, and delete their own documents. Deletion shall remove corresponding vectors and related metadata according to the documented retention policy.
- **FR-07 Question answering:** The system shall accept a question, retrieve relevant chunks only from the requesting user's authorized corpus, and generate an answer grounded in the retrieved context.
- **FR-08 Source citations:** Each answer shall include source document name, document ID, page or section where available, and a chunk/source reference.
- **FR-09 Insufficient evidence:** If retrieval produces insufficient evidence, the system shall state that it cannot answer from the available documents rather than inventing a factual answer.
- **FR-10 Retrieval options:** The system shall support configurable top-k retrieval and metadata filters. An advanced iteration shall add hybrid retrieval and reranking.
- **FR-11 Summarization:** Users shall be able to request a summary of a selected document they own, with references where feasible.
- **FR-12 Document comparison:** Users shall be able to select two authorized documents and receive a structured comparison grounded in both documents.
- **FR-13 Chat history:** The system shall store conversation messages and source references for the owning user and allow retrieval of that user's history.
- **FR-14 Evaluation:** The developer shall be able to run a repeatable evaluation dataset against a chosen pipeline configuration and export or inspect results.
- **FR-15 Observability:** The system shall record request/job IDs, status, latency, errors, retrieval settings, token usage and model/provider metadata where available. Secrets and full sensitive document contents shall not be logged.
- **FR-16 Frontend:** The UI shall provide sign-in, document upload/list/status, a chat interface, visible citations, and readable error/loading states.
- **FR-17 Rate and resource limits:** The API shall enforce configurable upload size, request limits, and model-call timeouts to protect service availability and cost.
- **FR-18 Tool workflow (advanced):** An advanced version may expose read-only tools for document search, summarization, and comparison through a controlled workflow. Tool calls shall enforce the same user authorization rules.

## 4. Key User Workflows

### 4.1 Upload and ingest

1. User signs in and uploads a supported file.
2. API validates the file and creates a document/job record.
3. Background worker extracts text, chunks it, generates embeddings, and writes vectors with ownership metadata.
4. The UI displays processing status and then success or a useful failure message.

### 4.2 Ask a question

1. User submits a question in a selected knowledge base.
2. Backend authenticates the user and applies ownership filters.
3. Retriever obtains relevant chunks; optional reranker reorders candidates.
4. LLM generates a grounded answer and citations.
5. API returns the answer, citations, and relevant request metadata.

### 4.3 Delete a document

1. User requests deletion of a document they own.
2. Backend verifies ownership.
3. System deletes document vectors and related records or marks deletion pending if asynchronous.
4. The UI confirms the final deletion status.

## 5. API Requirements (Initial Proposal)

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/auth/register` | Create user account |
| POST | `/auth/login` | Authenticate and return access token |
| POST | `/documents/upload` | Upload a PDF/DOCX and create an ingestion job |
| GET | `/documents` | List current user's documents with pagination |
| GET | `/documents/{document_id}` | Get document metadata and status |
| DELETE | `/documents/{document_id}` | Delete an owned document and its indexed chunks |
| GET | `/jobs/{job_id}` | Get ingestion job status |
| POST | `/chat` | Ask a question and return answer with citations |
| GET | `/conversations` | List current user's conversations |
| GET | `/conversations/{conversation_id}` | Get an owned conversation and messages |
| POST | `/documents/{document_id}/summarize` | Summarize an owned document |
| POST | `/documents/compare` | Compare two documents owned by the user |
| GET | `/health/live` | Liveness check |
| GET | `/health/ready` | Readiness check for required dependencies |

All protected endpoints must enforce ownership checks server-side. Use consistent error responses and appropriate HTTP status codes. Final request/response schemas should be defined in OpenAPI.

## 6. Data Requirements

| Entity | Suggested fields |
|---|---|
| User | id (UUID), email (unique), password_hash, created_at, is_active |
| Document | id (UUID), owner_id, filename, content_type, size_bytes, status, checksum, storage_key or safe file reference, error_summary, created_at, updated_at |
| IngestionJob | id (UUID), document_id, status, attempt_count, started_at, finished_at, error_code, created_at |
| DocumentChunk | id/reference, document_id, owner_id, chunk_index, text or source reference, page/section metadata, qdrant_point_id |
| Conversation | id (UUID), owner_id, title, created_at, updated_at |
| Message | id (UUID), conversation_id, role, content, citations JSONB, model_name, latency_ms, token_usage, created_at |
| EvaluationRun | id (UUID), configuration, dataset_version, aggregate metrics, started_at, finished_at |
| EvaluationCaseResult | id, run_id, case_id, retrieval results, answer metrics, failure notes |

Use PostgreSQL for transactional metadata and conversation history. Use Qdrant for vector points. Each vector payload must include document_id and owner_id (or an equivalent secure tenant key) so retrieval can be filtered by authorization. Avoid storing redundant sensitive text in multiple locations without a clear retention reason.

## 7. Non-Functional Requirements

| Area | Requirement |
|---|---|
| Security and privacy | Use password hashing, validated input, secrets in environment/secret storage, HTTPS in deployed environments, file validation, and authorization checks on every document operation. Treat uploaded documents as untrusted input. |
| Reliability | Handle dependency timeouts and transient errors. Use bounded retries with backoff for safe operations and avoid duplicate ingestion through idempotency/checksums. |
| Performance | For the local portfolio baseline, target API response under 2 seconds for metadata endpoints at light load. Document ingestion and LLM responses are asynchronous/variable and should be measured separately; establish realistic targets after benchmarking. |
| Scalability | Keep API and worker processes separable. Use background jobs for long-running ingestion and configure database/vector connection pools. |
| Maintainability | Use modular code, type hints, linting/formatting, migrations, configuration management, and documented setup steps. |
| Observability | Use structured logs, correlation IDs, health checks, and metrics for ingestion failures, retrieval latency, LLM latency, and token usage. |
| Accessibility/usability | Provide clear status messages, loading states, keyboard-accessible controls, and understandable errors. |
| Cost control | Set model timeouts, token/output limits, configurable model selection, and optional caching. Do not assume hosted model APIs are free. |

## 8. AI/RAG Quality Requirements

- The generation prompt must instruct the model to use retrieved context and identify when evidence is insufficient.
- Citations must refer to retrieved source chunks; citations must not be fabricated by the model.
- Build a versioned evaluation dataset with at least 50 questions, expected supporting documents/passages, and expected answer criteria.
- Measure retrieval quality (for example Recall@k or MRR where ground-truth relevance labels exist) and answer quality (for example faithfulness and answer relevance).
- Compare a baseline pipeline with at least one improvement such as hybrid search or reranking. Record the configuration and results.
- Review a sample of outputs manually. Automated LLM-based metrics are signals, not proof of correctness.
- Test unanswerable questions, conflicting documents, malformed files, empty extraction, prompt injection in documents, and cross-user retrieval attempts.

## 9. Acceptance Criteria

- A user can upload a valid PDF or DOCX and see ingestion status change to COMPLETED.
- A user can ask a question and receive an answer with at least one traceable citation when supporting evidence exists.
- For a question unsupported by the user's documents, the system returns a clear insufficient-evidence response.
- A user cannot retrieve, view, or delete another user's documents by changing an ID or query parameter.
- Deleting a document removes or invalidates its searchable vectors and associated access paths.
- Ingestion errors are visible as a failed status with a safe reason, and transient errors follow the configured retry limit.
- Automated tests cover core API, retrieval, ownership, and failure-handling paths.
- Evaluation runs can be repeated against a versioned dataset and their results can be compared.
- A new developer can run the application locally using documented setup steps and Docker Compose.
- The README includes architecture, setup, environment variables, API examples, known limitations, and measured evaluation results.

## 10. Suggested Repository Structure

```text
ai-knowledge-assistant/
├── backend/
│   ├── app/
│   │   ├── api/             # FastAPI routers
│   │   ├── core/            # Settings, auth, logging
│   │   ├── models/          # SQLAlchemy models
│   │   ├── schemas/         # Pydantic schemas
│   │   ├── services/        # Business logic
│   │   ├── ingestion/       # Parsing, chunking, embeddings
│   │   ├── retrieval/       # Search, filtering, reranking
│   │   ├── llm/             # Providers, prompts, generation
│   │   ├── workers/         # Background jobs
│   │   └── main.py
│   ├── tests/
│   ├── alembic/
│   └── pyproject.toml
├── frontend/
├── evaluation/
│   ├── datasets/
│   ├── run_evaluation.py
│   └── reports/
├── infra/
│   └── docker-compose.yml
├── .env.example
├── .gitignore
└── README.md
```

## 11. Implementation Constraints and Assumptions

- This is a learning and portfolio project, not a certified production system.
- Start with one LLM provider and one embedding model. Keep provider interfaces replaceable.
- Embedding model and vector dimensions must remain consistent within a Qdrant collection; changing models may require re-embedding documents.
- Use synthetic, public, or personally authorized documents for development. Do not upload employer-confidential data.
- Local development should be possible with Docker Compose; cloud deployment is optional.
- Costs, latency, file-size limits, retention periods, and model choices must be configurable and documented.

## 12. Definition of Done

Version 1 is complete when all core functional requirements (FR-01 through FR-10 and FR-15 through FR-17) are implemented, the acceptance criteria are demonstrated, tests pass, the app can be started from the README, and a reproducible evaluation report documents the baseline and at least one retrieval improvement. Summarization, comparison, and agent workflows may be delivered as later iterations.
