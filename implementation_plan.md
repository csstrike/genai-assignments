# Testcase Generator — Phase-Wise Implementation Plan

This plan operationalizes `my_testcase_generator_architecture.md` into sequential, buildable phases. Each phase produces a working, demonstrable increment and unlocks the next. Phases are designed to be executed in order; within a phase, workstreams can proceed in parallel.

---

## Guiding Principles

- Build the monorepo skeleton and shared contracts first (Section 16/17 of the architecture) so every later service reuses the same types, auth, logging, and config packages instead of re-inventing them.
- Get one connector + one source type end-to-end (ingestion → retrieval → generation) before adding breadth (more connectors, more formats).
- Defer expensive/complex pieces (LLM reranking, semantic dedupe, human-in-the-loop config, cloud deployment) until the core pipeline is proven with simpler substitutes.
- Each phase ends with a testable milestone, not just code.

---

## Phase 0 — Foundations & Repo Scaffolding

**Goal:** A running monorepo skeleton with no business logic yet, but all shared plumbing in place.

- Set up monorepo structure (`apps/`, `connectors/`, `processors/`, `packages/`, `prompts/`, `tests/`, `infra/`, `docs/`) per Section 16.
- Initialize `packages/shared-types`, `api-contracts`, `config`, `logging` packages.
- Set up TypeScript base config, linting, formatting, commit hooks.
- Provision local dev infra: MongoDB (local/dev), a local queue (e.g., Redis/RabbitMQ as a stand-in for the future cloud-managed queue).
- Set up CI pipeline skeleton (lint, build, unit test) in `infra/ci-cd`.
- Define the `NormalizedDocument` and `ChunkRecord` models (Section 6) in `packages/shared-types`.

**Milestone:** `npm/pnpm install && build` succeeds across all packages; CI runs green on an empty test suite; MongoDB and queue reachable from a local service.

---

## Phase 1 — Auth, API Gateway & User Management

**Goal:** Secure entry point and identity model before any business data flows through the system.

- Build `auth-service`: login, JWT issuance/refresh/logout, `/auth/me` (Section 14.1).
- Implement roles (`ADMIN`, `QA`, `DEVELOPER`, `VIEWER`) and RBAC data model (users, roles, projects collections).
- Build `api-gateway`: routing, JWT validation middleware, rate limiting, correlation/request IDs, API versioning (Section 4.1).
- Implement backend middleware chain: `authenticateJWT() -> authorizeRole() -> authorizeProjectAccess()` (Section 13).
- Implement User/Role APIs (Section 14.2).
- Add `packages/auth` shared package (JWT verify/sign, RBAC decorators/guards) so all downstream services reuse it.

**Milestone:** Can create a user, log in, receive a JWT, and call a protected placeholder route through the gateway with role enforcement verified by tests.

---

## Phase 2 — Source Connector Framework (Single Connector Proof)

**Goal:** Prove the generic connector contract end-to-end with exactly one simple source before scaling out.

- Implement `SourceConnector` interface (Section 4.3) in `packages/shared-types` / a `connectors` shared base.
- Build `source-service`: CRUD for source connections, connection test endpoint, sync trigger (Section 14.3).
- Implement the **first connector** (recommend Jira or a local file/PDF connector — pick the simplest to validate auth/fetch/normalize quickly) producing `NormalizedDocument` output.
- Persist source connection metadata (`source_connections` collection).

**Milestone:** Can register a source connection, test connectivity, and manually invoke `fetch()`/`incrementalFetch()` to retrieve raw records normalized into `NormalizedDocument` JSON.

---

## Phase 3 — Ingestion Pipeline (Job → Queue → Worker → Storage)

**Goal:** Full asynchronous ingestion pipeline for the one connector from Phase 2, ending in embedded chunks in MongoDB.

- Build `ingestion-service`: job creation, validation, job state machine (`PENDING → QUEUED → RUNNING → COMPLETED / PARTIAL_FAILURE / FAILED`), retry/cancel endpoints (Section 4.4, 14.4).
- Build `worker-service`: consume queue messages and execute `Fetch → Parse → Normalize → Chunk → Metadata enrichment → Embed → Persist` (Section 4.5, 8.1).
- Build one **format-specific processor** matching the Phase 2 connector (e.g., PDF or Jira normalization) in `processors/`.
- Implement chunking logic and content-hash generation (Section 6, 8.3 idempotency key: `sourceType + sourceId + sourceVersion + contentHash`).
- Build `embedding-service` wrapping Mistral AI (`packages/embedding`), storing `embeddingModel` in chunk metadata (Section 4.7).
- Implement `chunks` and `ingestion_jobs` MongoDB collections with indexes (`sourceType+sourceId`, `contentHash`, `updatedAt`) (Section 7).
- Implement incremental fetch support and dead-letter/retry handling for connector and embedding failures (Section 8.2, 22).

**Milestone:** Triggering an ingestion job for the Phase 2 source results in embedded, deduplicated chunks persisted in MongoDB, visible via `/ingestion/jobs/:id` and `/documents/:id/chunks`.

---

## Phase 4 — Retrieval Pipeline (Core, No Rerank/Dedupe Yet)

**Goal:** Basic hybrid retrieval working against real ingested data.

- Build `retrieval-service` with query preprocessing: normalization, abbreviation/synonym expansion, identifier extraction (Section 9.1).
- Implement BM25/keyword search against MongoDB text index.
- Implement vector search against MongoDB vector index.
- Implement a simple query-dependent router (start with a basic heuristic; refine later) (Section 9.2).
- Implement candidate merge/rank fusion (Section 9.3).
- Expose `/retrieval/search` API (Section 14.6).

**Milestone:** A query against ingested Phase 2/3 data returns a ranked, merged candidate set of chunks via `/retrieval/search`.

---

## Phase 5 — LLM Provider Abstraction

**Goal:** Configurable LLM access layer used by reranking, summarization, and generation.

- Build `llm-service` implementing the `LLMProvider` interface with adapters: `OpenAIProvider`, `GroqProvider`, `AnthropicProvider` (Section 4.9, 17).
- Implement provider/model configuration selection per use case (summarization, reranking, generation) (Section 14.10: `/llm/providers`, `/llm/config`).
- Add prompt versioning support (`prompts` collection, `/prompts` APIs).
- Add `packages/llm-abstraction` and `packages/prompt-engine` shared packages.

**Milestone:** Can switch active LLM provider/model via config and successfully call each adapter with a test prompt, with prompt versions stored and activatable.

---

## Phase 6 — Reranking, Deduplication & Summarization

**Goal:** Complete the retrieval pipeline per Section 9.4–9.6.

- Implement LLM-based reranking over a bounded candidate set (Section 9.4), exposed via `/retrieval/rerank`.
- Implement hybrid deduplication: exact/hash dedupe first, then semantic near-duplicate detection, with optional LLM adjudication for ambiguous pairs (Section 9.5), exposed via `/retrieval/deduplicate`.
- Implement LLM-based context summarization preserving requirement intent, acceptance criteria, API/UI behavior, constraints, defects, edge cases (Section 9.6), exposed via `/retrieval/summarize`.
- Wire preprocessing → routing → hybrid search → merge → rerank → dedupe → summarize into one pipeline callable end-to-end.

**Milestone:** Full retrieval pipeline (Section 15 subset) runs for a query and returns a compressed, deduplicated, reranked context ready for prompting.

---

## Phase 7 — Testcase Generation Service

**Goal:** Produce automation-ready testcases from retrieved context.

- Build `testcase-service`: prompt builder combining system prompt + query + summarized context + schema instructions (Section 10.1).
- Define and version the automation-ready testcase JSON schema (Section 10.2).
- Implement schema validation of LLM output.
- Implement generation categories support (positive/negative/boundary/etc.) (Section 10.3).
- Implement confidence classifier using deterministic rules (High/Medium/Low), not raw LLM self-reported confidence (Section 11).
- Persist to `generated_testcases` collection.
- Implement Testcase APIs: generate, list, get, patch, regenerate, validate, export, bulk-export (Section 14.7), and confidence API (Section 14.9).
- Wire the full REST flow from Section 15 (`/testcases/generate` end-to-end through gateway, auth, retrieval, LLM, schema validation, confidence, persistence).

**Milestone:** `POST /testcases/generate` returns a schema-valid, confidence-labeled, automation-ready testcase traceable to retrieved evidence.

---

## Phase 8 — Human-in-the-Loop Review Workflow

**Goal:** Configurable review process with full audit trail.

- Build `review-service`: approve/reject/modify/comment actions (Section 4.11, 12).
- Implement configurable review modes per project (`mandatory`, `optional`, `confidence_based` with threshold) (Section 12).
- Persist review audit trail (user, timestamp, original/modified version, comment, action) in `review_actions` collection.
- Implement Review APIs (Section 14.8).

**Milestone:** A generated testcase can move through `GENERATED → APPROVED/REJECTED/MODIFIED → APPROVED`, with full audit history retrievable via `/testcases/:id/review-history`.

---

## Phase 9 — Frontend (React UI)

**Goal:** Client-facing UI covering the full user journey.

- Scaffold `apps/frontend` (components, pages, hooks, services, store, types, routes per Section 16).
- Build screens: Source Connections, Ingestion Jobs, Query/Chat, Generated Testcases, Review/Approval, Export.
- Integrate JWT-based auth flow, role-aware UI gating.
- Integrate with all REST APIs built in Phases 1–8.

**Milestone:** A user can log in, connect a source, trigger ingestion, run a query, view generated testcases with confidence labels, review/approve them, and export results — entirely through the UI.

---

## Phase 10 — Connector & Format Breadth Expansion

**Goal:** Scale out from the single proven connector/format to the full supported input set.

- Add remaining connectors per Section 5/14.3: ADO, TestRail, Xray, Zephyr, Figma, Confluence/Wiki, release notes, Git repo, Swagger/OpenAPI, defect database.
- Add remaining format-specific processors: Excel, audio/video transcription, code parser, HTML/wiki extractor, structured record extractors (Section 4.6).
- Extract any complex/high-volume connector into an independent microservice where justified (Section 4.3, Q3).
- Add contract tests ensuring every new connector/processor conforms to the shared `Connector` contract (Section 17, 21).

**Milestone:** All source types listed in Section 5 can be connected, ingested, and made retrievable; contract tests pass for every connector.

---

## Phase 11 — Security Hardening

**Goal:** Close security gaps before production exposure.

- Enforce TLS everywhere; encrypt MongoDB traffic and at-rest data where supported.
- Move all secrets to a managed secret store; remove any inline credentials.
- Add file upload validation for Excel/PDF/recording inputs.
- Restrict connector permissions/scopes to minimum required.
- Add prompt-injection defenses: treat all retrieved content as untrusted data, never as instructions (Section 19 — "Important RAG security rule"). Add adversarial test cases (e.g., "ignore previous instructions") to the eval suite.
- Add rate limiting on public endpoints, audit logging of user actions.
- Run an OWASP-Top-10-oriented review across all services (auth, IDOR/project-access checks, input validation, injection, SSRF in connectors, dependency vulnerabilities).

**Milestone:** Security checklist in Section 19 fully satisfied; adversarial prompt-injection tests pass without leaking system instructions.

---

## Phase 12 — Observability

**Goal:** Instrument the system for operational visibility and RAG quality measurement.

- Add correlation IDs propagated across gateway → services → workers.
- Instrument ingestion metrics: jobs created/succeeded/failed, latency, documents/chunks processed, embedding latency/errors, duplicate rate.
- Instrument retrieval metrics: query latency, BM25/vector result counts, reranking latency, dedupe rate, context token count.
- Instrument generation metrics: LLM latency, token counts, generation/schema-validation failures, confidence distribution, approval/rejection/modification rates.
- Wire metrics to `/metrics`, `/health`, `/readiness` endpoints (Section 14.11) and a dashboard (e.g., Grafana/Cloud-native monitoring).

**Milestone:** Dashboards show live ingestion/retrieval/generation metrics; correlation ID can trace a single request across all services.

---

## Phase 13 — Full Testing & RAG Evaluation Suite

**Goal:** Comprehensive automated test coverage per Section 21.

- Unit tests: query preprocessing, connector adapters, chunking, hashing, dedupe, prompt construction, schema validation, confidence rules.
- Integration tests: MongoDB, queue, connector APIs, embedding provider, LLM adapters.
- Contract tests: all connectors and LLM adapters against their shared interfaces.
- End-to-end tests: create source → ingest → verify chunks → query → verify evidence → generate → review → export.
- Build a curated RAG evaluation dataset (query, expected sources, expected facts, expected testcase characteristics) and measure retrieval relevance, context precision/recall, duplicate rate, testcase correctness, hallucination rate, human approval rate (Section 21).

**Milestone:** CI runs the full test pyramid (unit/integration/contract/e2e) plus the RAG evaluation suite, with baseline metrics recorded for future regression comparison.

---

## Phase 14 — Cloud Deployment

**Goal:** Move from local/dev infra to public cloud per Section 18.

- Provision managed MongoDB, managed queue/event service, managed secrets manager.
- Containerize all services (`infra/docker`) and define Kubernetes manifests (`infra/kubernetes`) or chosen orchestration.
- Set up load balancer / API gateway ingress, private subnets for internal services.
- Set up centralized logging and managed metrics/tracing (`infra/monitoring`).
- Finalize CI/CD pipelines for automated build/test/deploy (`infra/ci-cd`).
- Run a staging deployment end-to-end smoke test, then promote to production.

**Milestone:** Full system running in the cloud, reachable through the load balancer, with managed MongoDB/queue/secrets, passing the same e2e smoke tests as local dev.

---

## Phase 15 — Extensions & Post-Launch Backlog

**Goal:** Track deliberately deferred/optional items for future iterations.

- Source relationship/traceability strategy (Q15 — explicitly skipped in architecture, revisit as an extension).
- Automatic LLM provider failover (deferred in Section 22 pending business justification).
- Rank-fusion weighting tuned experimentally per Section 9.2 (start heuristic, refine with data).
- Additional confidence-rule refinement based on production approval/rejection data.
- Expanded generation categories (security-oriented, data/state transition) hardening based on user feedback.

**Milestone:** Backlog is tracked with owners and acceptance criteria for future phases; not blocking initial production release.

---

## Summary Timeline View

```text
Phase 0  -> Repo & shared foundations
Phase 1  -> Auth + API Gateway
Phase 2  -> One connector, proven contract
Phase 3  -> Ingestion pipeline (job/queue/worker/embeddings)
Phase 4  -> Core hybrid retrieval (BM25 + vector)
Phase 5  -> LLM provider abstraction
Phase 6  -> Rerank + dedupe + summarization
Phase 7  -> Testcase generation + confidence
Phase 8  -> Human-in-the-loop review
Phase 9  -> React frontend (full journey)
Phase 10 -> Connector/format breadth expansion
Phase 11 -> Security hardening
Phase 12 -> Observability
Phase 13 -> Full test + RAG evaluation suite
Phase 14 -> Cloud deployment
Phase 15 -> Extensions & backlog
```

Phases 0–9 deliver a working single-source, single-format, full-journey MVP. Phases 10–14 harden and scale it into a production-grade, multi-source platform. Phase 15 captures explicitly deferred architectural decisions for future work.
