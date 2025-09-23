# LLM Chat Application - Initial Design

**Single-container Cloud Run backend + Vite/React frontend (two repos)**

## Table of Contents

1. [Use Cases](#1--use-cases)
2. [Confirmed Decisions](#2--confirmed-decisions)
3. [Architectural Overview](#3--architectural-overview)
4. [Infrastructure Setup](#4--infrastructure-setup)
5. [System Operations & Runtime Behavior](#5--system-operations--runtime-behavior)
6. [Frontend Implementation Specifications](#6--frontend-implementation-specifications)
7. [Backend Architecture & Implementation](#7--backend-architecture--implementation)
8. [Data Model & Persistence](#8--data-model--persistence)
9. [Testing Strategy (TDD + Integration)](#9--testing-strategy-tdd--integration)
10. [CI/CD (GitHub Actions) — Finalized Flow](#10--cicd-github-actions--finalized-flow)
11. [Migration Scripts & DB Priming](#11--migration-scripts--db-priming)
12. [Security & Hardening Checklist](#12--security--hardening-checklist)
13. [Observability & Metrics](#13--observability--metrics)
14. [Implementation Approach & Incremental Milestones](#14--implementation-approach--incremental-milestones)
15. [Prompt Structure Skeleton (For Later Generation)](#15--prompt-structure-skeleton-for-later-generation)
16. [Next Steps](#16--next-steps)

---

## 1 — Use Cases

### Primary Use Cases

1. **Multi-model AI Chat Interface**
   - Single unified interface to interact with multiple LLM providers (Claude, ChatGPT, Google AI, and local Gemma 2B)
   - Switch between models within the same thread (shared thread structure but separate conversation contexts and summaries per model)

2. **Private/Secure AI Conversations for Organizations**
   - Self-hosted solution for companies requiring data sovereignty and control
   - Keep sensitive conversations on-premises with local Gemma model
   - Encrypted storage of API keys and conversation data

3. **Cost-Optimized AI Usage**
   - Use free local Gemma 2B model for simple tasks
   - Switch to powerful cloud models (Claude, GPT-4) only when needed
   - Per-user API key management for cost attribution and tracking

4. **Persistent Conversation Management**
   - Long-term storage and organization of AI conversations in threads
   - Automatic summarization to maintain context in long conversations
   - Search and retrieve past conversations

5. **Team/Enterprise AI Platform**
   - Admin-controlled user management (no self-registration)
   - Centralized deployment with individual user accounts
   - Shared infrastructure with isolated user data

### Key Features Supporting These Use Cases

- **Per-user encrypted API key storage** - Each user manages their own API keys for different providers
- **Thread-based conversation organization** - Separate conversation contexts for different topics/projects
- **Automatic summarization** - Maintains context in long conversations (300-700 tokens)
- **Model-specific conversation contexts** - Each model maintains its own conversation history within a thread
- **Real-time streaming responses** - SSE-based streaming for responsive user experience
- **Local model support (Ollama/Gemma 2B)** - Run models locally for privacy-sensitive tasks
- **Health monitoring and status checks** - Real-time system health visibility in UI
- **Cursor-based pagination** - Efficiently handle large conversation histories
- **Flexible temperature controls** - Per-model temperature settings (sliders or toggles)
- **Inline async processing** - Synchronous title generation using local Ollama (3-word summary from first message) and async summarization triggered per API call (non-blocking)

---

## 2 — Confirmed Decisions

- **Two repos**: backend (Node.js + TypeScript + Express) and frontend (Vite + React)
- **Backend and Ollama** (Gemma 2 2B) will run together inside the same Cloud Run Docker container; Ollama runs as a separate process started by the container
- **Storage**: Firebase Realtime Database for both production and local/dev (using Firebase Emulator for local). DB provider selected by env var
- **Per-user API keys** (Claude, Google, ChatGPT) are stored encrypted in DB (encrypted with system key stored in Secret Manager/KMS)
- **No user registration/reset API**: an admin Cloud Function and bundled CLI utility to create/update users (password hash)
- **Model config** (available models, mapping to which API-key type, and allowed parameter constraints) is stored in DB and exposed via API for the frontend
- **Summarization & title generation** processed inline within API calls - synchronous title generation using local Ollama (3-word summary), summarization async but triggered per call (non-blocking response)
- **All CI/CD** done using GitHub Actions
- **Docker images** pushed to GitHub Container Registry (GHCR). Deploy to Cloud Run from GHCR
- **Frontend** is hosted on Firebase Hosting and deployed via GitHub Actions
- **Implementation approach**: TDD (tests drive implementation). Postman/Newman integration tests executed in build stage using in-memory DB / firebase emulator
- **No Redis** or external caching in initial design

---

## 3 — Architectural Overview

### High-Level Architecture

The application follows a **three-tier architecture**:
- **Frontend Layer**: React SPA with Vite bundler hosted on Firebase Hosting (CDN-distributed)
- **Backend API Layer**: Node.js/Express server running on Google Cloud Run (containerized, auto-scaling)
- **Data Layer**: Firebase Realtime Database (production with real database, development with Firebase Emulator)

**Integration Points:**
- RESTful API communication between frontend and backend
- Server-Sent Events (SSE) for real-time message streaming
- Title generation and summarization handled within API call flow (synchronous title generation using local Ollama for 3-word summary, summarization async non-blocking)

### System Components

#### Frontend Application (React SPA)
- User authentication & session management with JWT cookies
- Thread/conversation management UI with pagination
- Model selection & configuration interface
- Real-time message streaming with SSE client
- Health monitoring dashboard

#### Backend API Service (Node.js/Express)
- RESTful API endpoints for all application features
- JWT-based authentication/authorization middleware
- Model proxy layer that routes requests to appropriate LLM providers
- Inline processing for synchronous title generation using local Ollama (3-word summary) and async summarization (non-blocking)
- Health check and metrics endpoints

#### LLM Integration Layer
- **External providers**: Claude API, OpenAI API, Google AI API using encrypted user API keys
- **Local model**: Ollama with Gemma 2B running in-container for privacy-sensitive tasks
- Unified interface abstracting provider differences
- Streaming response handling for all models

#### Data Persistence Layer
- **Primary**: Firebase Realtime Database with real-time sync capabilities
- **Local Development**: Firebase Emulator Suite for offline development
- Encrypted API key storage using KMS-managed system keys
- Thread, message, and user data management with cursor-based pagination

---

## 4 — Infrastructure Setup

### Backend Infrastructure

- **Deployment Platform**: Google Cloud Run (containerized, auto-scaling)
- **Container Image**: Docker image with Node.js runtime and Express server
- **Resource Configuration**: CPU and memory sized for concurrent request handling
- **Scaling**: Auto-scaling with scale-to-zero capability
- **Inline Processing**: Synchronous title generation using local Ollama and async summarization handled within API request context (no separate workers)

#### Ollama Runtime (Local Model)

**Container Configuration:**
- The Cloud Run container image includes the Ollama binary or supervisor script
- Model files can be packaged in the image or downloaded at runtime to ephemeral filesystem

**Model Loading Behavior:**
- Models are not pre-loaded at container start
- API endpoint `/api/models/:modelId/load` triggers model loading into Ollama runtime
- Each Cloud Run instance must load the model independently (instance-local)

**Tradeoffs & Constraints:**
- Cloud Run instances are ephemeral and may scale-to-zero
- Model load latency and memory usage must be accepted per instance
- Large model files increase image size; consider fetching from Cloud Storage at startup
- Document resource sizing carefully (CPU, memory requirements)

### Frontend Infrastructure

- **Hosting Platform**: Firebase Hosting with global CDN distribution
- **Build**: Vite-bundled React SPA with optimized production builds
- **Deployment**: Automated via GitHub Actions on merge to main
- **Static Assets**: Cached at edge locations for low latency

### Database Infrastructure

- **Production**: Firebase Realtime Database with automatic scaling
- **Development**: Firebase Emulator Suite for local testing
- **Data Persistence**: Real-time sync with offline support
- **Security**: Encrypted storage for sensitive data (API keys)
- **Backup Strategy**: Regular automated backups via Firebase console

### Local Development & Test Environment

- Provide docker-compose for local development:
  - Backend container built from Dockerfile; runs server and can optionally spawn Ollama locally (if developer has installed/available)
  - For local dev use Firebase Emulator Suite
- Use Firebase Emulator Suite for DB & auth locally
- Provide local seed and migration scripts to prime DB for dev
- Provide `scripts/add-user` CLI which can be run locally or as Cloud Function in production

---

## 5 — System Operations & Runtime Behavior

### Model Loading & Ollama Lifecycle

#### API behavior

**`POST /api/models/gemma-2b/load`:**
- If model already loaded on this instance → return 200 with `loaded=true`
- If not loaded:
  - Start loader (spawn Ollama process if not running; download model artifacts if needed)
  - Record status in DB `models_config[gemma-2b].runtime_status` (or a per-instance status store — we may store only global status plus instance-specific ephemeral status)
  - Return job id and initial status, or keep connection open and stream progress updates

**`GET /api/models/gemma-2b/status`:**
- Returns current status and estimated readiness

**Loader implementation:**
- Either spawn ollama as a child process in container or call the ollama binary
- Keep an internal process supervisor in container to restart on failure
- Expose process logs to the streaming SSE so UI can display progress if possible

#### UI:

- Selecting Gemma model triggers `GET models/:id/status` and shows loading | loaded | not_loaded
- If `not_loaded`, show a "Load" button. On click call `POST /api/models/:id/load`
- Display progress bar if backend returns progress updates; otherwise show loading spinner and a loaded indicator when done

#### Notes on scaling & cost

Each new Cloud Run instance must load local model on demand. Document cost + latency. This is acceptable per requirement but needs planning for instance sizing and concurrency settings (e.g., set concurrency=1 for model-heavy instances or configure memory accordingly).

### Model Catalog & Configuration

Store full model catalog in `models_config` in DB. Example fields relevant to UI:
- `modelId`: e.g., `gemma-2b-local`, `claude-opus-4.1`, `gpt-4o`
- `provider`: `ollama|claude|google|openai`
- `apiKeyType`: `user_claude`, `user_google`, `user_openai`, or `none` (local)
- `temperatureType`: `range` (min, max, step) or `enum` (e.g., [0,1]) — used by UI to render slider or fixed options
- `requires_local_load`: boolean (true for local Gemma)
- `display_name`, `priority`, `notes`

#### UI behavior:

- Model dropdown is populated from the `/api/models` API (reads `models_config`)
- If `temperatureType` is enum with [0,1] model, the UI shows a dropdown or toggle instead of a continuous slider
- For local Gemma (`requires_local_load=true`), UI shows load status next to model and a "Load model" button when not loaded. On load, show progress / status
- If user hasn't provided a required API key for selected provider, the model appears disabled with a CTA to set API key in profile

### Prompt Processing & Inline Operations

#### Prompt Assembly (per message)

1. Fetch `user.profile.system_prompt`
2. Fetch `thread.models[modelId].summary` (if present)
3. Fetch most recent messages for the (thread,model) in reverse chronological order until the configured `CONTEXT_TOKEN_BUDGET`
4. Append the new user message
5. Send to model with streaming enabled if requested
6. After assistant reply completes, trigger async summarization (non-blocking) for that (thread,model)

#### Summarization Process

- Summarization triggered asynchronously per API call but doesn't block response
- After each assistant completion, async summarization begins using the same model
- Summaries are ~300–700 tokens per requirement
- Updates `threads.models[modelId].summary` asynchronously

#### Title Generation

- Title generation happens synchronously during thread creation using local Ollama
- Generates 3-word summary from first user message content
- Completed within the API response for thread creation (after async LLM call initiation)
- Uses local Ollama for fast processing without external API dependency

#### Inline Processing

- Title generation occurs synchronously using local Ollama during thread creation (3-word summary)
- Summarization triggered asynchronously per API call (non-blocking)
- No external queue systems or workers - all processing within API context
- Manual summarization available via dedicated endpoint for user control
- Summaries displayed in UI under respective thread/model conversations

### Pagination & Data Loading

- All listing endpoints support cursor-based pagination:
  - Request: `GET /api/threads?limit=20&cursor=<cursor>`
  - Response: `{ items: [...], nextCursor: "abc", hasMore: true }`
- Messages endpoint supports `limit` and `cursor` for infinite scroll. Frontend uses intersection observer to fetch next page on scroll
- Search in `GET /api/threads?q=...` implemented server-side (title + model summaries)

### Edge Cases & Operational Notes

- **Model loading failures**: surface clear error to user; persist failure reason in DB; allow retry
- **Partial responses**: mark assistant message as partial and allow client to request fill or retry
- **Scaling/Growth**: note that per-instance model loads can multiply resource consumption; document recommended concurrency / maxInstances settings for Cloud Run
- **Backpressure**: restrict concurrent model calls per user to protect resources

### Database Watcher Timeouts

- **Message Generation**: 30 second timeout for LLM response generation
- **Model Loading**: 300 second (5 minute) timeout for local model loading operations
- **Summary Generation**: 60 second timeout for async summarization tasks
- **Health Checks**: 5 second timeout for external service health verification
- **Connection Cleanup**: Automatic cleanup of orphaned watchers after client disconnect
- **Resource Limits**: Maximum 100 concurrent watchers per backend instance
- **Watcher Deregistration**: DB watchers are deregistered when message is complete, SSE connection closes, or timeout occurs
- **Resource Management**: Explicit cleanup prevents memory leaks and ensures proper resource management

### Health Monitoring & Version Display

#### Health Check API

`/api/health` returns:
- `service`: `ok|degraded|down`
- `uptime_seconds`
- `db`: `{ ok: true/false, provider: firebase|firebase-emulator, latency_ms }`
- `ollama`: `{ status: not_loaded|loading|loaded|error, model: gemma-2b, progress?: 0..100 }`
- `internal_api`: `{ status: ok }`

#### Version Display

- Frontend displays both backend version (from `/api/version`) and frontend version (from `VITE_APP_VERSION`) in the topbar
- Backend version endpoint: `GET /api/version` returns `{ version, build_time, commit }`
- Version information helps track deployed builds and troubleshoot issues
- Health status shown near version with refresh button that calls `/api/health` and updates UI

### User Management (Admin-only)

Since there is no self-registration, user accounts are created and managed exclusively through administrative tools:

#### Admin Cloud Function
- Deployed as a separate Cloud Function with admin-only access controls
- Accepts: `username`, `password`, optional `display_name`
- Performs password hashing (bcrypt or similar)
- Writes user record to database with hashed password
- Can update existing user passwords or profile information
- Protected by Cloud IAM - only admins can invoke

#### CLI Utility (`scripts/add-user`)
- Bundled script for local development and production admin use
- Same functionality as Cloud Function but runs locally
- Connects directly to database (Firebase or emulator)
- Usage: `npm run add-user -- --username john@example.com --password <password> --display-name "John Doe"`
- Can be run in production with appropriate Firebase credentials

#### Security Considerations
- No password reset flow - admins manually reset via Cloud Function or CLI
- No user self-service - prevents unauthorized account creation
- All user management operations logged for audit trail
- Password complexity requirements enforced by admin tools

---

## 6 — Frontend Implementation Specifications

**See:** [frontend-implementation.md](./frontend-implementation.md)

This section covers the complete frontend architecture including React component structure, Material-UI integration, state management with Zustand, detailed page specifications, responsive design strategy, and performance optimizations.

---

## 7 — Backend Architecture & Implementation

**See:** [backend-implementation.md](./backend-implementation.md)

This section covers the complete backend architecture including Node.js/Express service design, authentication with JWT, model integration layer, real-time communication via SSE, comprehensive API endpoint specifications, security implementation, and performance optimizations.

---

## 8 — Data Model & Persistence

**See:** [data-model-persistence.md](./data-model-persistence.md)

This section covers the complete data model architecture including detailed database schema definitions, relationships, indexing strategies, migration system design with versioning support, data validation and constraints, performance optimizations, security and encryption implementation, backup and recovery procedures, and database provider abstraction layer.

---


## 9 — Testing Strategy (TDD + Integration)

### Unit tests (run in CI):
- Prompt builder, token estimation, encryption utilities, small service functions
- Use Jest (Node) and run in CI

### Integration tests:
- Postman collection (or Newman) that exercises auth, thread creation, message flow, summarization trigger, and health endpoints
- These integration tests are executed during CI build stage:
  - Start backend in test mode inside the job (use `npm start:test`), connecting to an in-memory DB:
    - Use Firebase Emulator Suite for local testing
    - For Firebase: use Firebase Emulator Suite or mock DB adapter (in-memory implementation) to avoid external dependency
  - Run Newman (postman) collection against the started server

### E2E (separate pipeline):
- Playwright/Cypress to run a small happy-path with mocked or test backend

### TDD workflow:
- Developers write failing tests (unit or integration) then implement code to make tests pass
- CI ensures no regression by running unit + integration tests before build/publish

---

## 10 — CI/CD (GitHub Actions) — Finalized Flow

### Common points

- **Trigger**: push to main → create a release tag (default patch bump) and start pipeline OR `workflow_dispatch` to allow manual bump major/minor
- **Tagging**: `vMAJOR.MINOR.PATCH` created by GH Actions earlier in the workflow
- **Secrets required** in GitHub Actions:
  - `GHCR_PAT` or `GITHUB_TOKEN` for GHCR push
  - `GCP_SA_KEY` or use GitHub Workload Identity / OIDC + service account for Cloud Run deploy
  - `FIREBASE_TOKEN` for frontend deploy
  - `SECRET_MANAGER_KEY` details for test and deploy if needed to decrypt test system key

### Backend workflow (merge->main)

1. Checkout repo
2. Bump version (default patch) and create tag `vX.Y.Z` (or accept provided tag)
3. Install deps, run linter
4. Run unit tests
5. Start backend in test mode with in-memory DB (Firebase Emulator Suite)
6. Run Newman Postman integration tests against started test server
7. Build Docker image with `--build-arg VERSION=${VERSION}` and label
8. Push image to GHCR (`ghcr.io/org/backend:${VERSION}`)
9. Deploy to Cloud Run service referencing GHCR image (gcloud auth with SA or via OIDC)
10. On success, publish GitHub Release with notes and artifacts as needed

### Frontend workflow (merge->main)

1. Checkout repo
2. Bump version (independent repo) default patch and create tag
3. Install deps, lint, run unit tests
4. Start backend-mock or backend-stub as needed for integration tests if required
5. Build vite with `VITE_APP_VERSION=${VERSION}` env var
6. Run any automated UI integration tests against built output (optional)
7. Deploy to Firebase Hosting using `firebase deploy` with `FIREBASE_TOKEN`
8. Publish GitHub Release

**Note**: Integration tests should run before build/publish steps to prevent releasing broken code.

---

## 11 — Migration Scripts & DB Priming

- Provide a `migrations/` folder with numbered migration files (JS) that run in sequence
- Migration runner executes at build or on startup (controlled via env var `RUN_MIGRATIONS=true`) and can:
  - Create initial `models_config` entries
  - Create default system nodes
  - Create seed data for local dev
- Support commands:
  - `migrate:up`, `migrate:down`, `migrate:status`
- For Firebase, use an adapter that applies changes safely (idempotent operations), and for Mongo use migration tool (e.g., migrate-mongo or standalone runner)
- On system start with `RUN_MIGRATIONS=true`, the backend runs pending migrations before listening to requests

---

## 12 — Security & Hardening Checklist

### Authentication:
- JWT tokens with strong signing key stored in Secret Manager (rotate periodically)
- Use httpOnly + Secure cookies for auth by default

### Encryption:
- Encrypt per-user API keys before storing in DB using KMS-managed system key

### Network:
- Limit inbound for Cloud Run via IAM; Cloud Run private egress to access other systems

### Input validation & Sanitization:
- Sanitize Markdown rendered on frontend and server. Strip dangerous tags and attributes
- Validate request payloads thoroughly; use schema validation (Zod/Joi)

### Rate limiting:
- Implement per-user and per-IP rate limits in backend (in-memory limiter for now)

### Least-privilege:
- Service accounts for Cloud Run have minimal permissions (only Firebase Admin, KMS decrypt if necessary)

### Secrets:
- Use Secret Manager for system secrets. Do not store raw secrets in DB or commit to repo

### Logging:
- Structured logs with request_id correlation; redact sensitive info

### Dependency management:
- Use automated Dependabot or similar to keep dependencies up-to-date

### CI security:
- Use GitHub OIDC to avoid long-lived cloud credentials

### Vulnerability scanning:
- Run container image vulnerability scan in CI before push/publish (optional GitHub Action)
- Document all above steps and runbook for incident response

---

## 13 — Observability & Metrics

- Expose metrics endpoint (Prometheus style or JSON) for:
  - Requests per endpoint, model calls, tokens consumed, worker queue depth
- Logs to Cloud Logging with structured fields
- Health endpoint used by Uptime checks and displayed in UI

---


## 14 — Implementation Approach & Incremental Milestones

This section describes milestone-by-milestone plans for the backend and frontend repos. Each milestone is TDD-first: write failing tests (unit / integration), implement code until tests pass, expand Postman/Newman integration tests, and ensure CI runs green.

**Two important process-level rules:**
1. The backend and frontend are separate repos. There must be a clearly documented, reproducible mechanism that a coding agent (or CI job) can use to start the backend service (test-mode or container) and keep it available for frontend integration/E2E tests
2. Frontend CI/CD + deployment must be created in the first milestone of the frontend flow (not the last milestone)

### Backend Flow (Backend API + Ollama process inside single container)

#### Milestone 0 — Repo scaffolding, minimal APIs, CI skeleton, and coding-agent start mechanism

**Objectives:**
- Create repo scaffolding: `src/`, `tests/`, `migrations/`, `scripts/`, `docs/`
- Add base Dockerfile and a `docker-compose.yml` for local development (with Firebase Emulator)
- Provide environment config conventions and `.env.example`
- Implement minimal mock endpoints: `GET /api/health`, `GET /api/models` (returns []), `GET /api/version` (reads build arg)

**Coding-agent start mechanism:**
- Provide an explicit start/test command that a coding agent can run to start the backend in test-mode with an in-memory DB:
  - Example design-level script names: `npm run start:test` and `npm run start:ci`
  - `start:test` runs the server bound to `0.0.0.0:${PORT:-3000}` and uses Firebase Emulator Suite. It prints readiness and exposes `/api/health`
  - `start:ci` is equivalent but optimized for CI: deterministic ports, logs to stdout, no interactive prompts
- Provide `docker-compose.ci.yml` which can start the API container (built locally) and optional DB container for testing; CI can `docker-compose -f docker-compose.ci.yml up --build -d` to start the service
- Provide a `wait-for-health.sh` script (or equivalent behavior) that polls `/api/health` until it returns ok or times out — CI/test runners and coding agents should use this before running integration tests

**TDD:**
- Add failing unit tests for placeholder endpoints

**CI (GitHub Actions) skeleton:**
- Lint + unit tests
- Job step that builds Docker image (locally) but does not push
- A job step template that shows how to run `start:ci` or `docker-compose.ci.yml`, run Newman/Postman tests, and tear down

**Acceptance:**
- A coding agent or developer can start the backend with a single documented command for local dev and CI
- Health probe works and is usable by test runners

#### Milestone 1 — DB adapter & initial migrations

**Tasks:**
- Implement DB-adapter abstraction (Firebase / Mongo) and migrations runner
- Seed `models_config` via migrations
- Ensure `start:test` uses migrations to load seed data into in-memory DB so integration tests have deterministic data

**Acceptance:**
- Migrations runnable locally and in CI; integration tests can rely on seeded `models_config`

#### Milestone 2 — Auth & user utilities (incl. Cloud Function / script)

**Tasks:**
- Implement `createOrUpdateUser` utility accessible as:
  - CLI script `scripts/add-user` for local dev (invokable by coding agent)
  - Cloud Function (production) for administrative invocation
- Auth endpoints and JWT issuance; `start:test` should support creating test admin user automatically or via the CLI

**Acceptance:**
- Coding agent can create a test user and authenticate against the running test server

#### Milestone 3 — Models & loader APIs

**Tasks:**
- Implement `GET /api/models`, `GET /api/models/:id/status`, `POST /api/models/:id/load`
- `POST /api/models/:id/load` triggers in-container Ollama load. For test-mode, this endpoint can simulate load progress (so integration tests do not hang)

**Acceptance:**
- Frontend tests can request model catalog and simulate model-load interactions in CI

#### Milestone 4 — Threads, messages, pagination

**Tasks:**
- Messages storage in separate collection with pagination
- Thread CRUD APIs and message POST/GET with cursors
- `start:test` must be able to populate test threads/messages for deterministic frontend tests

**Acceptance:**
- Frontend integration tests use seeded threads/messages

#### Milestone 5 — Prompt builder, streaming proxy & summarization enqueue

**Tasks:**
- Implement prompt builder and call-proxy logic (local or remote)
- SSE streaming for `POST /api/threads/:id/models/:model/messages`
- Trigger async summarization (non-blocking) and track in operations collection

**Acceptance:**
- Streaming API exercised by integration tests (can be mocked or actual local Ollama if the CI environment supports it)

#### Milestone 6 — Ollama loader implementation in-container

**Tasks:**
- Implement supervisor logic to spawn the Ollama process on demand (on `POST /api/models/gemma-2b/load`)
- In CI/test-mode, loader can be stubbed or run in a lightweight mode to avoid excessive resource needs

**Acceptance:**
- Loader endpoint works in local dev; CI can stub or run a minimal loader

#### Milestone 7 — Inline title generation & async summarization

**Tasks:**
- Implement synchronous title generation using local Ollama (3-word summary from first message)
- Implement async summarization triggered per API call (non-blocking)
- Provide manual summarization endpoint for user-initiated summary regeneration
- Implement DB watcher deregistration on message completion or connection close
- In test-mode, operations complete synchronously for deterministic testing

**Acceptance:**
- Summaries and titles can be generated in test environment deterministically for integration tests
- DB watchers properly deregister to prevent resource leaks

#### Milestone 8 — Health, metrics, security

**Tasks:**
- Finalize health endpoint with DB & Ollama checks
- Harden API (validation, rate limiting)

**Acceptance:**
- `/api/health` returns clear status usable by frontend tests and readiness checks

#### Milestone 9 — CI/CD, image build & deploy to GHCR and Cloud Run

**Tasks:**
- Finalize GitHub Actions:
  - On merge to main, create release tag, run tests (unit + Newman integration against `start:ci`), build Docker image embedding version, push to GHCR, deploy to Cloud Run
- Ensure CI steps include `start:ci` or `docker-compose.ci.yml` pattern so the same way the coding agent started backend is used in CI

**Acceptance:**
- CI validates integration tests that require a running backend

#### Milestone 10 — Admin tools, docs & runbook

**Tasks:**
- Provide `scripts/add-user`, `scripts/migrate`, `scripts/start:test`, `docker-compose.ci.yml`, and documented guidance for coding agents and CI

**Acceptance:**
- Enough tooling exists for automated agents to run full test suites locally and in CI

### Frontend Flow (Vite + React) — with CI/CD & deploy in Milestone 0

#### Milestone 0 — Repo scaffolding, CI/CD & initial deploy capability (MUST-HAVE)

**Important change**: CI/CD and deployment are part of the first milestone.

**Tasks:**
- Create frontend repo scaffold: `src/`, `tests/`, `public/`, `docs/`
- Add basic Vite + React + TypeScript setup with Tailwind
- Create GitHub Actions workflows now (milestone 0) that:
  - Run lint + unit tests
  - Build the app with `VITE_APP_VERSION` injected (versioning workflow)
  - Deploy to Firebase Hosting (use `FIREBASE_TOKEN` secret); workflow includes steps to:
    - Start a backend test service before running integration tests
    - Run integration tests and E2E only after backend readiness
- Provide a frontend CI job pattern that can:
  - Start backend service (for integration) using documented methods
  - Run Newman/E2E tests against the started backend
  - Deploy to Firebase Hosting on success

**Coding-agent integration mechanism (for UI tests):**

Document and provide two ways for coding agent / CI to start a test backend to run UI integration tests:

1. **Direct Node start**: call `npm -w backend start:ci` or run in backend repo `npm run start:ci` (recommended when backend source is available to the agent). This starts a local server using in-memory DB. The frontend CI job can call this if both repos are checked out in the same workspace or the backend repo is a submodule or available via HTTP download.

2. **Docker Compose**: invoke `docker-compose -f docker-compose.ci.yml up --build -d` (which brings up the backend image and optional DB container). Then `wait-for-health.sh` polls `/api/health` until ready. This is the recommended approach when the agent builds the backend image as part of a combined CI workflow or uses a container from GHCR.

- Provide `scripts/frontend/run-integration-tests.sh` (documented) that:
  - Accepts `BACKEND_URL` env var (defaults to `http://localhost:3000`)
  - Waits for `BACKEND_URL/api/health` to be ready
  - Runs Newman or Playwright tests against `BACKEND_URL`
- Create initial placeholder pages (Login, Dashboard) and fail-first unit tests (TDD)

**Acceptance:**
- Frontend CI can start a backend test service using either `start:ci` (node) or `docker-compose.ci.yml`, run integration tests against it, and deploy to Firebase Hosting if tests pass

#### Milestone 1 — Auth & app skeleton

**Tasks:**
- Implement login page, token handling with httpOnly cookie
- Ensure frontend has env config to point to test backend (e.g., `VITE_BACKEND_URL`)
- Add unit tests for auth UI
- CI must run these tests and block deployment on failure

**Acceptance:**
- Login works in integration tests with the started backend

#### Milestone 2 — Threads UI & pagination

**Tasks:**
- Implement sidebar with paged loads (`GET /api/threads` with cursor)
- Integration tests (Playwright/Newman) that create threads on the started backend and validate UI reflects them

**Acceptance:**
- Frontend integration tests perform calls against the running backend started by CI/agent

#### Milestone 3 — Models UI & loader interactions

**Tasks:**
- Populate model dropdown from `GET /api/models`
- Implement temperature controls and model-load flow; when selecting Gemma model, frontend calls `POST /api/models/gemma-2b/load` and polls `GET /api/models/gemma-2b/status`
- Integration tests simulate model load progress (backend test-mode should support simulated progress)

**Acceptance:**
- Frontend integration tests can exercise model-load interactions while the backend is running in CI

#### Milestone 4 — Chat pane & streaming

**Tasks:**
- Implement SSE client to receive streaming tokens from backend
- Integration tests verify streaming behavior: start a streaming message post and ensure tokens arrive incrementally (backend test-mode can simulate streaming)

**Acceptance:**
- Streaming UI validated via CI/E2E

#### Milestone 5 — Profile, API keys UI & other flows

**Tasks:**
- Profile screen to add API keys and edit `system_prompt`; tests to verify UI flows and that missing API keys disable relevant models

**Acceptance:**
- Integration coverage for profile flows

#### Milestone 6 — Health UI + deploy polish

**Tasks:**
- Health indicator in topbar reads `/api/health`
- Version display uses `VITE_APP_VERSION` and a button to fetch backend `/api/version`
- Finalize Firebase Hosting deploy step in CI (already present from milestone 0) and any post-deploy checks

**Acceptance:**
- Full CI pipeline builds, tests (integration against running backend), and deploys

#### Milestone 7 — Accessibility, monitoring & final E2E

**Tasks:**
- Accessibility checks, final E2E happy-path tests
- Post-deploy smoke test to validate deployment

**Acceptance:**
- All E2E tests pass and deployment smoke checks succeed

### Cross-repo Integration & Agent Usage Patterns

How automated agent / CI should start backend for frontend tests:

1. **When both repos are available in CI job** (monorepo or combined pipeline):
   - CI job builds backend, runs `npm run start:ci` in background or `docker-compose -f docker-compose.ci.yml up -d` and then runs `scripts/frontend/run-integration-tests.sh` with `BACKEND_URL` set to local address
   - Use wait-for-health to ensure readiness

2. **When frontend CI cannot build backend** (separate pipelines):
   - Frontend CI can pull a test-ready backend Docker image from GHCR with a `:test` tag (produced by backend CI) and run via `docker run` or `docker-compose`
   - Poll `/api/health` until ready

3. **Agent-driven local dev / coding-agent (Jules) usage**:
   - Use `npm run start:test` (fast mode with in-memory DB) to get quick local server with seeded data for UI tests
   - For heavier local runs, run `docker-compose up` to simulate production-like environment

4. **Readiness & stability**:
   - The wait-for-health behavior must check `db` and `ollama` keys in `/api/health`. For frontend integration tests, the minimal requirement is `service: ok` and `db.ok: true`. If the test needs local Gemma, `ollama.status` must be `loaded` or test-mode must simulate the loaded state

### Acceptance Criteria for Coding-agent/Start Mechanism (Non-negotiable)

- A single documented command exists to start backend in test-mode (no external DB required) and return readiness via `/api/health`
- The command works both locally and in CI job environment
- The frontend CI workflow uses the documented mechanism to launch a backend instance prior to running integration/E2E tests
- The backend provides `start:ci` and `start:test` scripts and a `docker-compose.ci.yml` to facilitate agent usage
- All integration tests rely only on the provided start mechanism and seeded data (no manual steps)

### TDD Enforcement & CI Gating

- Each milestone must provide failing tests first (unit or integration)
- CI will block merges if unit or integration tests (Newman/Postman) fail against the started backend instance
- The GitHub Actions job must orchestrate backend startup, health polling, test execution, build, and deploy steps in the order described — same pattern used by coding agents

---

## 15 — Prompt Structure Skeleton (For Later Generation)

This is the template we will use later to generate Jules/engineer prompts for each repo and per milestone. We are not generating prompt content now — only the structure/sections that each prompt should contain.

### Common prompt structure (for both backend & frontend repo prompts)

1. **Title** — short statement of the task (repo + milestone)
2. **Context** — one-paragraph summary of the project and design decisions (links to design doc)
3. **Goal / Objective** — what must be delivered by the task
4. **Scope** (must have / nice-to-have) — exact acceptance criteria; TDD expectations
5. **Tech Stack** — runtime, libraries, versions (Node LTS, Express, Vite, React)
6. **APIs / Interfaces** — list of endpoints (method, path, auth, request/response sketch)
7. **Data model** — collections/tables and key fields
8. **Testing requirements** — unit tests to write first (describe test cases), integration tests (Postman collection items), CI integration
9. **CI/CD** — actions, build, test, tag & publish steps the prompt should create
10. **Deployment instructions** — container build args, runtime env variables, secrets consumed (list)
11. **Migration & seed** — migration tasks to include and seed data to prime
12. **Observability & logging** — required logs and metrics to emit
13. **Security checklist** — how to handle secrets, encryption, cookie/JWT rules
14. **Developer ergonomics** — scripts to include (start:test, migrate, add-user)
15. **Acceptance tests** — exact Postman tests or unit tests that must pass before merge
16. **Deliverables** — files/folders to be present, README items, docs pages to create
17. **Incremental plan** — tasks & subtasks for the milestone
18. **Notes & constraints** — any limitations or special considerations (e.g., model loading latency)
19. **Versioning & release** — how the prompt should ensure versioning is embedded into build

We will reuse this structure when you ask us to generate the actual repo-specific prompt content for each milestone.

---

## 16 — Next Steps

This design is finalized and ready. When you confirm, I will:

1. Generate the backend prompt (following the structure above) for Phase 0 and Phase 1 (scaffold + core CRUD/TDD)
2. Generate the frontend prompt (Phase 0 + Phase 1) thereafter
3. Generate sample GitHub Actions workflows and skeleton migrations/ files and Postman collection schema as part of the prompts