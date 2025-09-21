# LLM Chat Application - Initial Design

**Single-container Cloud Run backend + Vite/React frontend (two repos)**

---

## 1 — Use Cases

### Primary Use Cases

1. **Multi-model AI Chat Interface**
   - Single unified interface to interact with multiple LLM providers (Claude, ChatGPT, Google AI, and local Gemma 2B)
   - Switch between models within the same thread (shared thread structure and summary, but separate conversation contexts per model)

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
- **Inline async processing** - Title generation on first message (within API call) and async summarization triggered per API call (non-blocking)

---

## 2 — Confirmed Decisions

- **Two repos**: backend (Node.js + TypeScript + Express) and frontend (Vite + React)
- **Backend and Ollama** (Gemma 2 2B) will run together inside the same Cloud Run Docker container; Ollama runs as a separate process started by the container
- **Storage**: Firebase Realtime Database for both production and local/dev (using Firebase Emulator for local). DB provider selected by env var
- **Per-user API keys** (Claude, Google, ChatGPT) are stored encrypted in DB (encrypted with system key stored in Secret Manager/KMS)
- **No user registration/reset API**: an admin Cloud Function and bundled CLI utility to create/update users (password hash)
- **Model config** (available models, mapping to which API-key type, and allowed parameter constraints) is stored in DB and exposed via API for the frontend
- **Summarization & title generation** processed inline within API calls - title on first message, summarization async but triggered per call (non-blocking response)
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
- Title generation and summarization handled within API call flow (title synchronous on first message, summarization async non-blocking)

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
- Inline processing for title generation (first message) and async summarization (non-blocking)
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
- **Inline Processing**: Title generation and summarization handled within API request context (no separate workers)

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
2. Fetch `thread.models[modelId].last_summary` (if present)
3. Fetch most recent messages for the (thread,model) in reverse chronological order until the configured `CONTEXT_TOKEN_BUDGET`
4. Append the new user message
5. Send to model with streaming enabled if requested
6. After assistant reply completes, trigger async summarization (non-blocking) for that (thread,model)

#### Summarization Process

- Summarization triggered asynchronously per API call but doesn't block response
- After each assistant completion, async summarization begins using the same model
- Summaries are ~300–700 tokens per requirement
- Updates `threads.models[modelId].last_summary` asynchronously

#### Title Generation

- Title generation happens synchronously on first user message in thread (across all models)
- Uses same model with a short instruction and writes `thread.title` if default
- Completed within the API response for first message

#### Inline Processing

- Title generation occurs synchronously on first message in a thread
- Summarization triggered asynchronously per API call (non-blocking)
- No external queue systems or workers - all processing within API context
- Manual summarization available via dedicated endpoint for user control
- Summaries displayed in UI under respective thread/model conversations

### Pagination & Data Loading

- All listing endpoints support cursor-based pagination:
  - Request: `GET /api/threads?limit=20&cursor=<cursor>`
  - Response: `{ items: [...], nextCursor: "abc", hasMore: true }`
- Messages endpoint supports `limit` and `cursor` for infinite scroll. Frontend uses intersection observer to fetch next page on scroll
- Search in `GET /api/threads?q=...` implemented server-side (title + last_summary)

### Edge Cases & Operational Notes

- **Model loading failures**: surface clear error to user; persist failure reason in DB; allow retry
- **Partial responses**: mark assistant message as partial and allow client to request fill or retry
- **Scaling/Growth**: note that per-instance model loads can multiply resource consumption; document recommended concurrency / maxInstances settings for Cloud Run
- **Backpressure**: restrict concurrent model calls per user to protect resources

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

### Frontend Architecture & Component Hierarchy

The frontend follows a **component-driven architecture** using React with TypeScript, organized in layers:

#### Application Structure
```
src/
├── components/          # Reusable UI components
├── pages/              # Route-level page components
├── hooks/              # Custom React hooks
├── services/           # API client and external services
├── store/              # State management (Zustand/Context)
├── types/              # TypeScript type definitions
├── utils/              # Helper functions and utilities
└── styles/             # Global styles and theme
```

#### State Management Strategy
- **Authentication State**: JWT token, user session, login status
- **Thread Management**: Active thread, thread list, pagination cursors
- **Message Handling**: Streaming messages, conversation history, model contexts
- **Model Configuration**: Available models, loading states, temperature settings
- **UI State**: Sidebar collapse, modal states, notification toasts
- **Health Monitoring**: System status, version info, real-time health checks

### Navigation Structure & Routes

The application uses **React Router** with protected routes and role-based access:

```typescript
/                          → Dashboard (authenticated users only)
/login                     → Login Page (unauthenticated only)
/threads/:threadId         → Thread View (authenticated)
/profile                   → Profile Settings (authenticated)
/profile/api-keys          → API Key Management (authenticated)
/admin                     → Admin Panel (admin role only)
/health                    → Health Monitor (authenticated)
/404                       → Not Found Page
```

### Detailed Page Specifications

#### 1. Login Page (`/login`)
**Layout Structure:**
```
┌─────────────────────────────────────────┐
│ [App Logo]           LLM Chat Platform  │
├─────────────────────────────────────────┤
│                                         │
│    ┌─────────────────────────────┐      │
│    │     Login Form Box          │      │
│    │  ┌─────────────────────────┐│      │
│    │  │ Username/Email          ││      │
│    │  │ [input field]           ││      │
│    │  └─────────────────────────┘│      │
│    │  ┌─────────────────────────┐│      │
│    │  │ Password                ││      │
│    │  │ [password field]        ││      │
│    │  └─────────────────────────┘│      │
│    │  [Remember Me] □            │      │
│    │  [Login Button]             │      │
│    │  [Error Message Area]       │      │
│    └─────────────────────────────┘      │
│                                         │
│    Health Status: ● Online              │
│    Version: Frontend v1.0.0             │
└─────────────────────────────────────────┘
```

**Components:**
- `LoginForm`: Username/password inputs with validation
- `HealthIndicator`: Shows backend connectivity status
- `VersionDisplay`: Frontend and backend version info
- `ErrorAlert`: Authentication error messages

**Functionality:**
- Form validation (client-side)
- JWT token handling via httpOnly cookies
- Redirect to dashboard on successful auth
- Health check display for system status
- Responsive design for mobile/tablet

#### 2. Dashboard/Main Chat Interface (`/`)
**Layout Structure:**
```
┌─────────────────-───────────────────────────────────────────---─┐
│ [≡] LLM Chat    [Health ●] [Profile] [v1.0.0] [Logout]          │
├────────────────-─┬──────────────────────────────────────────---─┤
│ Thread Sidebar   │          Main Chat Area                      │
│ ┌────────────-─┐ │ ┌─────────────────────────────────────────-┐ │
│ │[+ New Thread]│ │ │ Thread: "Project Planning Discussion"    │ │
│ └─────────────-┘ │ │ ┌─────────────────────────────────────┐  │ │
│ ┌────────────-─  │ │ │ Model Selector & Controls           │  │ │
│ │Thread 1      │ │ │ │ [Claude Opus ▼] [Temp: ██□□] [Load] │  │ │
│ │"Design..."   │ │ │ └─────────────────────────────────────┘  │ │
│ │● Claude ● GPT│ │ │                                          │ │
│ └────────────-─┘ │ │ Chat Messages Area:                      │ │
│ ┌─────────────┐  │ │ ┌────────────────────────────────────-─┐ │ │
│ │Thread 2     │  │ │ │ User: "Help me plan the architecture"│ │ │
│ │"Code..."    │  │ │ │ ⏰ 2:30 PM                           │ │ │
│ │● Local ●    │  │ │ └───────────────────────────────────-──┘ │ │
│ └─────────────┘  │ │ ┌────────────────────────────────────-─┐ │ │
│ [Load More...]   │ │ │ Assistant: "I'll help you create..." │ │ │
│                  │ │ │ [Streaming indicator: ▋]             │ │ │
│ Search Box:      │ │ │ ⏰ 2:31 PM                           │ │ │
│ [🔍 Search...]   │ │ └───────────────────────────────────-──┘ │ │
└────────────────-─┤ │ ┌────────────────────────────────────-─┐ │ │
                   │ │ │ Message Input Area                   │ │ │
                   │ │ │ [Type your message here...]          │ │ │
                   │ │ │ [📎 Attach] [🎤 Voice] [➤ Send]     │ │ │
                   │ │ └───────────────────────────────────-──┘ │ │
                   │ └───────────────────────────────────────-──┘ │
                   └─────────────────────────────────────────---──┘
```

**Main Components:**
- `HeaderBar`: Navigation, health, profile, version display
- `ThreadSidebar`: Thread list with search and pagination
- `ChatArea`: Active conversation display
- `ModelSelector`: Model dropdown with load status
- `TemperatureControl`: Slider or toggle based on model configuration
- `MessageList`: Scrollable conversation history
- `MessageInput`: Text input with send controls
- `StreamingIndicator`: Real-time typing indicator

**Sidebar Details:**
- Thread cards show title, creation time, active models (colored dots)
- Search functionality filters by title and summary
- Infinite scroll with cursor-based pagination
- "New Thread" button creates thread and redirects
- Collapsible on mobile/tablet

**Chat Area Details:**
- Model selector populated from `/api/models`
- Temperature control adapts to model configuration (slider vs toggle)
- Load button appears for local models when not loaded
- Message timestamps and status indicators
- Copy message buttons
- Markdown rendering for assistant responses

#### 3. Thread View (`/threads/:threadId`)
**Layout Structure:**
```
┌─────────────────────────────────────────────────────────────┐
│ [←] Thread: "System Architecture Planning"  [Share] [⚙️]    │
├─────────────────────────────────────────────────────────────┤
│ Model Tabs: [Claude Opus ●] [GPT-4 ●] [Gemma Local ○]      │
├─────────────────────────────────────────────────────────────┤
│ Summary Panel (Collapsible):                                │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 📋 Thread Summary (Claude Opus)                         │ │
│ │ "Discussion covers microservice architecture patterns..." │ │
│ │ Last updated: 5 min ago  [Regenerate Summary]           │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ Message History:                                            │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 👤 User (to Claude Opus) - 2:15 PM                     │ │
│ │ "What are the best practices for microservices?"        │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🤖 Claude Opus - 2:15 PM                               │ │
│ │ "Here are the key microservice patterns..."             │ │
│ │ [Copy] [Retry] [Continue]                               │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 👤 User (to GPT-4) - 2:20 PM                           │ │
│ │ "Compare this to monolithic architecture"               │ │
│ └─────────────────────────────────────────────────────────┘ │
│ [Load Earlier Messages...]                                  │
├─────────────────────────────────────────────────────────────┤
│ Active Model: [Claude Opus ▼] [Temp: ██████░░] [Settings]  │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ [Type message for Claude Opus...]                       │ │
│ │ [📎] [🎤] [➤]                                           │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Components:**
- `ThreadHeader`: Navigation, title editing, sharing options
- `ModelTabs`: Switch between model conversations in thread
- `SummaryPanel`: Collapsible summary display per model
- `MessageTimeline`: Chronological message display with model context
- `MessageActions`: Copy, retry, continue generation
- `ModelContextSwitcher`: Change active model for new messages

**Functionality:**
- Each model maintains separate conversation context within thread
- Summaries update asynchronously and display under each model tab
- Message history shows which model each message was sent to
- Model switching preserves input text
- Infinite scroll for message history
- Title editing with auto-save

#### 4. Profile Settings (`/profile`)
**Layout Structure:**
```
┌─────────────────────────────────────────────────────────────┐
│ [←] Profile Settings                                        │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 👤 User Information                                     │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Display Name: [John Developer              ] [Save] │ │ │
│ │ │ Username: john@company.com (read-only)              │ │ │
│ │ │ Member since: January 2024                          │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🤖 Default AI Settings                                 │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Default Model: [Claude Opus ▼]                     │ │ │
│ │ │ Default Temperature: [██████░░░░] 0.7               │ │ │
│ │ │ Streaming Enabled: ☑️                              │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 📝 System Prompt Configuration                         │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ [Large text area for system prompt]                 │ │ │
│ │ │ "You are a helpful AI assistant..."                 │ │ │
│ │ │                                                     │ │ │
│ │ │ Character count: 245/2000                           │ │ │
│ │ │ [Reset to Default] [Save Changes]                   │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🔑 API Keys Management                                  │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Claude API Key: ●●●●●●●●●●sk-xxx [Configured] [Edit]│ │ │
│ │ │ OpenAI API Key: [Not configured] [Add Key]          │ │ │
│ │ │ Google AI Key: ●●●●●●●●●●key-xxx [Configured] [Edit]│ │ │
│ │ │                                                     │ │ │
│ │ │ ⚠️ API keys are encrypted and stored securely       │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Components:**
- `UserInfoCard`: Display name editing, read-only username
- `DefaultSettingsCard`: Model and temperature preferences
- `SystemPromptEditor`: Large text area with character counting
- `APIKeyManager`: Masked key display with configuration status

#### 5. API Key Management (`/profile/api-keys`)
**Layout Structure:**
```
┌─────────────────────────────────────────────────────────────┐
│ [←] API Key Management                                      │
├─────────────────────────────────────────────────────────────┤
│ ℹ️ Secure API Key Storage                                   │
│ Your API keys are encrypted with enterprise-grade security  │
│ and never stored in plain text.                            │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🔵 Claude API Configuration                             │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Status: ✅ Configured and Valid                     │ │ │
│ │ │ Key Preview: ●●●●●●●●●●sk-ant-xxx                   │ │ │
│ │ │ Last Updated: 3 days ago                            │ │ │
│ │ │ [Update Key] [Test Connection] [Remove]             │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🟡 OpenAI API Configuration                            │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Status: ❌ Not Configured                           │ │ │
│ │ │ Required for: GPT-4, GPT-3.5 models                │ │ │
│ │ │ [Add OpenAI API Key]                                │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🔴 Google AI API Configuration                         │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Status: ⚠️ Key Invalid (check required)              │ │ │
│ │ │ Key Preview: ●●●●●●●●●●key-xxx                      │ │ │
│ │ │ Error: Authentication failed (401)                  │ │ │
│ │ │ [Update Key] [Test Connection] [Remove]             │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ Modal Example (when "Add Key" clicked):                    │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Add OpenAI API Key                                 [✕]  │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ API Key: [sk-xxxxxxxxxxxxxxxx] [Show/Hide]         │ │ │
│ │ │ ⚠️ Keep your API key secure and never share it      │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ │ [Test & Save] [Cancel]                                  │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Components:**
- `APIKeyCard`: Per-provider key management with status
- `KeyInputModal`: Secure key entry with validation
- `ConnectionTester`: Verify API key validity
- `SecurityNotice`: Encryption and security information

#### 6. Admin Panel (`/admin`) - Admin Role Only
**Layout Structure:**
```
┌─────────────────────────────────────────────────────────────┐
│ [←] Admin Panel                                             │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 👥 User Management                                      │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Search: [🔍 Search users...]                        │ │ │
│ │ │ [+ Add New User]                                    │ │ │
│ │ │                                                     │ │ │
│ │ │ ┌─────────────────────────────────────────────────┐ │ │ │
│ │ │ │ john@company.com | John Dev | Active | [Edit]  │ │ │ │
│ │ │ │ Last login: 2 hours ago                         │ │ │ │
│ │ │ └─────────────────────────────────────────────────┘ │ │ │
│ │ │ ┌─────────────────────────────────────────────────┐ │ │ │
│ │ │ │ alice@company.com | Alice | Active | [Edit]     │ │ │ │
│ │ │ │ Last login: 1 day ago                           │ │ │ │
│ │ │ └─────────────────────────────────────────────────┘ │ │ │
│ │ │ [Load More Users...]                                │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🤖 Model Configuration                                  │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Available Models:                                   │ │ │
│ │ │ ┌─────────────────────────────────────────────────┐ │ │ │
│ │ │ │ Claude Opus | Claude | ✅ Active | [Configure] │ │ │ │
│ │ │ │ Max Context: 200k tokens                        │ │ │ │
│ │ │ └─────────────────────────────────────────────────┘ │ │ │
│ │ │ ┌─────────────────────────────────────────────────┐ │ │ │
│ │ │ │ Gemma 2B | Local | ⚠️ Loading | [Manage]        │ │ │ │
│ │ │ │ Status: Loading on 2/3 instances                │ │ │ │
│ │ │ └─────────────────────────────────────────────────┘ │ │ │
│ │ │ [+ Add Model Configuration]                         │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 📊 System Metrics                                       │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Active Users: 12 | Total Threads: 245 | Messages: 1.2k│ │ │
│ │ │ API Calls (24h): 1,547 | Errors: 3                 │ │ │
│ │ │ Storage Used: 15.3 MB | Model Load Time: 45s avg    │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Components:**
- `UserManagementTable`: User CRUD operations with search
- `ModelConfigurationPanel`: Model availability and settings
- `SystemMetricsDisplay`: Usage statistics and performance data
- `UserEditModal`: Create/edit user accounts

#### 7. Health Monitor (`/health`)
**Layout Structure:**
```
┌─────────────────────────────────────────────────────────────┐
│ [←] System Health Monitor                     [🔄 Refresh]  │
├─────────────────────────────────────────────────────────────┤
│ Overall Status: ✅ All Systems Operational                   │
│ Last Updated: 2 seconds ago                                 │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🖥️ Backend Service                                      │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Status: ✅ Healthy                                  │ │ │
│ │ │ Uptime: 2 days, 14 hours                           │ │ │
│ │ │ Version: v1.2.3 (commit: abc123f)                  │ │ │
│ │ │ Response Time: 42ms                                 │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🗄️ Database Connection                                  │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Status: ✅ Connected                                 │ │ │
│ │ │ Provider: Firebase Realtime Database               │ │ │
│ │ │ Latency: 15ms                                       │ │ │
│ │ │ Last Query: 1 second ago                            │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🤖 Local AI Model (Ollama)                             │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Status: ✅ Loaded and Ready                         │ │ │
│ │ │ Model: Gemma 2B                                     │ │ │
│ │ │ Memory Usage: 3.2 GB / 8 GB                        │ │ │
│ │ │ Last Request: 5 minutes ago                         │ │ │
│ │ │ Average Response Time: 1.2s                         │ │ │
│ │ │ [Force Reload Model]                                │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 🔗 External API Status                                  │ │
│ │ ┌─────────────────────────────────────────────────────┐ │ │
│ │ │ Claude API: ✅ Operational (last check: 30s ago)    │ │ │
│ │ │ OpenAI API: ✅ Operational (last check: 30s ago)    │ │ │
│ │ │ Google AI: ⚠️ Degraded Performance (150ms avg)      │ │ │
│ │ └─────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Components:**
- `HealthOverview`: System-wide status indicator
- `ServiceStatusCard`: Individual service health monitoring
- `DatabaseStatusCard`: DB connection and performance metrics
- `ModelStatusCard`: Local model loading and resource usage
- `ExternalAPIStatus`: Third-party service availability

#### 8. 404 Not Found (`/404` or unmatched routes)
**Layout Structure:**
```
┌─────────────────────────────────────────────────────────────┐
│ [🏠] LLM Chat Platform                                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                    🤖 404 - Not Found                      │
│                                                             │
│             The page you're looking for doesn't exist.     │
│                                                             │
│                  [🏠 Return to Dashboard]                   │
│                  [📱 Contact Support]                       │
│                                                             │
│                                                             │
│                Helpful Links:                               │
│                • Dashboard                                  │
│                • Profile Settings                          │
│                • Health Monitor                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Component Library Specifications

#### Core Reusable Components

**Navigation Components:**
- `HeaderBar`: Main navigation with user menu, health indicator
- `Sidebar`: Collapsible navigation panel
- `Breadcrumbs`: Page hierarchy navigation

**Data Display Components:**
- `ThreadCard`: Thread summary with model indicators
- `MessageBubble`: Individual message display with actions
- `StatusIndicator`: Health/loading status with colors
- `VersionBadge`: Version information display

**Input Components:**
- `ModelSelector`: Dropdown with search and filtering
- `TemperatureSlider`: Dynamic slider or toggle based on model
- `MessageInput`: Rich text input with attachment support
- `SearchInput`: Debounced search with suggestions

**Feedback Components:**
- `LoadingSpinner`: Various loading states
- `ProgressBar`: Model loading and operation progress
- `ToastNotification`: Success/error messages
- `ConfirmationModal`: Action confirmation dialogs

**Layout Components:**
- `PageContainer`: Standard page wrapper with consistent spacing
- `ContentGrid`: Responsive grid system
- `SplitPane`: Resizable sidebar layout
- `ScrollContainer`: Infinite scroll implementation

### State Management Approach

#### Zustand Store Structure
```typescript
interface AppState {
  // Authentication
  auth: {
    user: User | null;
    token: string | null;
    isAuthenticated: boolean;
    login: (credentials: LoginCredentials) => Promise<void>;
    logout: () => void;
  };

  // Threads Management
  threads: {
    items: Thread[];
    activeThreadId: string | null;
    isLoading: boolean;
    hasMore: boolean;
    cursor: string | null;
    fetchThreads: (search?: string) => Promise<void>;
    createThread: (title?: string) => Promise<Thread>;
    deleteThread: (id: string) => Promise<void>;
  };

  // Messages & Streaming
  messages: {
    byThreadAndModel: Record<string, Message[]>;
    isStreaming: boolean;
    streamingMessageId: string | null;
    sendMessage: (threadId: string, modelId: string, content: string) => Promise<void>;
    appendStreamingToken: (token: string) => void;
  };

  // Models Configuration
  models: {
    available: ModelConfig[];
    loadingStates: Record<string, LoadingState>;
    loadModel: (modelId: string) => Promise<void>;
    fetchModels: () => Promise<void>;
  };

  // UI State
  ui: {
    sidebarCollapsed: boolean;
    activeModal: string | null;
    notifications: ToastMessage[];
    toggleSidebar: () => void;
    showModal: (type: string, props?: any) => void;
    addNotification: (message: ToastMessage) => void;
  };

  // Health Monitoring
  health: {
    status: SystemHealth | null;
    lastCheck: Date | null;
    checkHealth: () => Promise<void>;
  };
}
```

### User Interaction Flows

#### Thread Creation & Message Flow
1. User clicks "New Thread" in sidebar
2. `createThread()` API call creates thread with default title
3. User redirected to `/threads/:threadId`
4. Model selector populated from available models
5. User types message and selects model
6. `sendMessage()` initiates streaming response
7. Real-time token updates via SSE
8. Message saved and thread updated

#### Model Loading Flow
1. User selects local model (Gemma) from dropdown
2. `GET /api/models/gemma-2b/status` checks current state
3. If not loaded, "Load Model" button appears
4. User clicks load, `POST /api/models/gemma-2b/load` called
5. Progress bar shows loading status via polling or SSE
6. Model becomes available for conversation

#### API Key Configuration Flow
1. User navigates to Profile → API Keys
2. Click "Add Key" for specific provider
3. Modal opens with secure input field
4. Key entered and validated via test connection
5. If valid, key encrypted and stored via `POST /api/profile/api-keys`
6. Models requiring that provider become available

### Responsive Design Strategy

#### Breakpoint System
- **Mobile**: < 640px (single column, collapsible sidebar)
- **Tablet**: 640px - 1024px (sidebar overlay, compact controls)
- **Desktop**: > 1024px (full layout with sidebar)

#### Mobile Adaptations
- Sidebar becomes slide-over modal
- Message input moves to bottom fixed position
- Model selector becomes bottom sheet
- Chat area uses full viewport height
- Touch-optimized button sizes (44px minimum)

#### Tablet Adaptations
- Sidebar width reduces to 280px
- Model controls stack vertically
- Temperature slider becomes larger touch target
- Thread cards use horizontal layout

#### Accessibility Features
- ARIA labels for all interactive elements
- Keyboard navigation support
- Focus management for modals and dropdowns
- Screen reader announcements for streaming messages
- High contrast mode support
- Reduced motion preferences

### Performance Optimizations

#### Code Splitting
- Route-based code splitting for each page
- Lazy loading of admin panel components
- Dynamic imports for heavy dependencies

#### State Optimization
- Virtualized scrolling for large message lists
- Debounced search input (300ms)
- Memoized expensive computations
- Selective re-renders with React.memo

#### Network Optimization
- Request deduplication for model status checks
- Optimistic updates for message sending
- Background prefetching of thread data
- Compression for large message payloads

---

## 7 — Data Model & Persistence

### Top-level DB layout (Firebase JSON / Mongo collections)

- **`users` collection** (or node): user profile, pwd hash, display data, encrypted API keys, default model settings
- **`threads` collection**:
  - `id`, `user_id`, `title`, `created_at`, `updated_at`, `meta…`
  - `models` (dictionary or separate collection reference): for each supported model id, there is model-specific metadata (e.g., `modelId`, `loaded_status`, `last_summary`, `updated_at`)
- **`messages` collection** (separate collection):
  - Each message: `id`, `thread_id`, `model_id`, `role` (user|assistant|system), `content`, `tokens`, `status` (partial|complete|failed), `created_at`, `updated_at`
  - Pagination-friendly (indexed by `thread_id` + `created_at`)
- **`models_config` collection** (persisted model catalog):
  - Each model: `modelId`, `provider` (ollama|claude|google|openai), `displayName`, `apiKeyType` (which user API key to use), `temperatureOptions` (allowed values or min/max/step or enumerated options), `maxContextTokens`, `notes`
- **`operations` collection** (async operation tracking):
  - Track async operations: `operationId`, `type` (summarize), `thread_id`, `model_id`, `status`, `result`, `created_at`, `updated_at`
- **`system` node**:
  - version metadata baked into the container; `app_start_time` etc. (but versions are packaged into the build — not written to DB by CI)

### DB provider abstraction

- Backend must implement a data-layer abstraction to switch between Firebase Realtime DB (production) and Firebase Emulator (local) based on `DB_PROVIDER` env var
- Migration scripts must support both backends (provide a unified migration API that runs JS migration files and can operate on either provider)

---


## 8 — API Endpoints (Finalized Surface)

### Auth & meta

- `POST /api/login` — body `{ username, password }` → returns JWT set as httpOnly cookie (or token)
- `POST /api/logout` — clears cookie
- `GET /api/health` — Public. Returns JSON with: `status`, `uptime`, `db` (OK/ERR), `db_details`, `ollama` (not_loaded | loading | loaded | error), `internal_api_ok`
- `GET /api/version` — Auth required. Return container baked version info `{ version, build_time, commit }`

### Profile & API keys

- `GET /api/profile` — Auth required. Returns profile metadata (no raw keys)
- `PATCH /api/profile` — update profile fields (`display_name`, `system_prompt`, `default_model`, `default_temp`)
- `POST /api/profile/api-keys` — Auth required. Body `{ claude?, google?, openai? }` — backend encrypts & stores

### Models & catalog

- `GET /api/models` — Auth required. Return model catalog from DB
- `GET /api/models/:modelId/status` — Auth required. Returns `loaded_status` and other runtime metrics for local model (if applicable)
- `POST /api/models/:modelId/load` — Auth required. Triggers local Ollama load for gemma-2b and returns job id / immediate status. Will stream status updates via SSE or return job id to poll

### Threads & messages

- `POST /api/threads` — create thread `{ title? }`
- `GET /api/threads` — Auth required. Supports `q` (search), `limit`, `cursor` (cursor-based pagination). Return summary, last message snippet, per-model indicators
- `GET /api/threads/:threadId` — metadata for thread (no message payloads by default)
- `DELETE /api/threads/:threadId` — Auth required
- `GET /api/threads/:threadId/models/:modelId/messages` — Auth required. Supports `limit`, `cursor` (pagination), returns messages ordered by `created_at`
- `POST /api/threads/:threadId/models/:modelId/messages` — Auth required. Body `{ content, temperature?, stream?: boolean }`. Behavior:
  - Save user message
  - Build prompt using user system prompt + model-specific thread summary + recent messages under token budget
  - Call model (local Ollama if ollama provider; otherwise provider using decrypted user API key)
  - If `stream=true`, return SSE stream of tokens; else wait for final reply
  - Save assistant message as partial then complete
  - Trigger async summarization (non-blocking) for that (thread,model)
- `PATCH /api/threads/:threadId/title` — auth required update title
- `POST /api/threads/:threadId/models/:modelId/summarize` — User or admin endpoint to manually trigger/regenerate summary. Returns the summary text which can be displayed in UI under the thread

### Admin & utilities (invoked as Cloud Function / CLI)

- Cloud Function `createOrUpdateUser` — accepts `username`, `password`, optional `display_name` — writes hashed passwd to DB
- CLI `scripts/add-user` for local dev (same behavior)

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
- Implement title generation on first message (synchronous within API call)
- Implement async summarization triggered per API call (non-blocking)
- Provide manual summarization endpoint for user-initiated summary regeneration
- In test-mode, operations complete synchronously for deterministic testing

**Acceptance:**
- Summaries and titles can be generated in test environment deterministically for integration tests

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