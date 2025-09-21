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

The frontend follows a **component-driven architecture** using React with TypeScript and Material-UI, organized in layers:

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
└── styles/             # Material-UI theme and global styles
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
/404                       → Not Found Page
```

### Detailed Page Specifications

The frontend consists of 5 main pages, focusing on core functionality with a streamlined interface.

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
│    │  [Login Button]             │      │
│    │  [Error Message Area]       │      │
│    └─────────────────────────────┘      │
│                                         │
├─────────────────────────────────────────┤
│ Health Status: ● Online                 │
│ Frontend: v1.0.0 | Backend: v1.2.3     │
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
┌─────────────────────────────────────────────────────────────┐
│ [≡] LLM Chat                        Main Chat Area         │
├─────────────────┬───────────────────────────────────────────┤
│ Left Sidebar    │          Chat Interface                   │
│ ┌─────────────┐ │ ┌─────────────────────────────────────────┐ │
│ │[+ New Thread]│ │ │ Thread: "Project Planning Discussion"   │ │
│ └─────────────┘ │ │ ┌─────────────────────────────────────┐ │ │
│ ┌─────────────┐ │ │ │ Model Selector & Controls           │ │ │
│ │Thread 1     │ │ │ │ [Claude Opus ▼] [Temp: ██□□] [Load] │ │ │
│ │"Design..."  │ │ │ └─────────────────────────────────────┘ │ │
│ │● Claude ● GPT│ │ │                                         │ │
│ └─────────────┘ │ │ Chat Messages Area:                     │ │
│ ┌─────────────┐ │ │ ┌─────────────────────────────────────┐ │ │
│ │Thread 2     │ │ │ │ User: "Help me plan the architecture"│ │ │
│ │"Code..."    │ │ │ │ ⏰ 2:30 PM                          │ │ │
│ │● Local ●    │ │ │ └─────────────────────────────────────┘ │ │
│ └─────────────┘ │ │ ┌─────────────────────────────────────┐ │ │
│ [Load More...]  │ │ │ Assistant: "I'll help you create..." │ │ │
│                 │ │ │ [Streaming indicator: ▋]             │ │ │
│ Search Box:     │ │ │ ⏰ 2:31 PM                          │ │ │
│ [🔍 Search...]  │ │ └─────────────────────────────────────┘ │ │
│                 │ │ ┌─────────────────────────────────────┐ │ │
│ ═══════════════ │ │ │ Message Input Area                   │ │ │
│ System Status:  │ │ │ [Type your message here...]          │ │ │
│ Backend: ● OK   │ │ │ [📎 Attach] [🎤 Voice] [➤ Send]      │ │ │
│ Database: ● OK  │ │ └─────────────────────────────────────┘ │ │
│ Ollama: ● Ready │ └─────────────────────────────────────────┘ │
│                 │                                           │
│ ═══════════════ │                                           │
│ [Profile ▼]     │                                           │
│  • Settings     │                                           │
│  • Logout       │                                           │
│                 │                                           │
│ Version Info:   │                                           │
│ Frontend v1.0.0 │                                           │
│ Backend v1.2.3  │                                           │
└─────────────────┴───────────────────────────────────────────┘
```

**Main Components:**
- `AppHeader`: Simple navigation header with app name and collapsible trigger
- `LeftSidebar`: Integrated sidebar with threads, health, profile, and version info
- `ChatArea`: Active conversation display on the right side
- `ModelSelector`: Model dropdown with load status
- `TemperatureControl`: Slider or toggle based on model configuration
- `MessageList`: Scrollable conversation history
- `MessageInput`: Text input with send controls
- `StreamingIndicator`: Real-time typing indicator
- `SystemHealthStatus`: Health indicators for backend, database, and Ollama
- `ProfileDropdown`: Profile menu with settings and logout options

**Left Sidebar Details:**
- Thread cards show title, creation time, active models (colored dots)
- Search functionality filters by title and summary
- Infinite scroll with cursor-based pagination
- "New Thread" button creates thread and redirects
- System health status (Backend/Database/Ollama) with green/red indicators
- Profile dropdown menu with "Settings" and "Logout" options
- Version information display (Frontend and Backend versions)
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

#### 5. 404 Not Found (`/404` or unmatched routes)
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

#### Material-UI Based Components

**Theme Configuration:**
- **Primary Color**: Clear blue theme for brand consistency
- **Typography**: Material-UI typography scale with clear, readable fonts
- **Spacing**: Consistent 8px grid system
- **Color Palette**: Light theme with clear contrast ratios for accessibility

**Navigation Components:**
- `AppBar` (MUI): Simple header with app name and sidebar toggle
- `LeftSidebar`: Custom collapsible sidebar with MUI components
- `Drawer` (MUI): Mobile sidebar implementation

**Data Display Components:**
- `ThreadCard`: Custom card using MUI Card component
- `MessageBubble`: Custom component with MUI Paper and Typography
- `Chip` (MUI): Status indicators for health/loading states
- `Badge` (MUI): Version and notification indicators

**Input Components:**
- `Autocomplete` (MUI): Model selector with search and filtering
- `Slider` (MUI): Temperature control slider
- `TextField` (MUI): Message input and form fields
- `FormControl` (MUI): Structured form inputs

**Feedback Components:**
- `CircularProgress` (MUI): Loading indicators
- `LinearProgress` (MUI): Model loading progress bars
- `Snackbar` (MUI): Success/error toast notifications
- `Dialog` (MUI): Modal dialogs and confirmations

**Layout Components:**
- `Container` (MUI): Page wrapper with responsive breakpoints
- `Grid` (MUI): Responsive grid system
- `Box` (MUI): Flexible layout container
- `Stack` (MUI): Vertical/horizontal component stacking

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
    profileMenuOpen: boolean;
    toggleSidebar: () => void;
    showModal: (type: string, props?: any) => void;
    addNotification: (message: ToastMessage) => void;
    toggleProfileMenu: () => void;
  };

  // Health Monitoring (integrated into main dashboard)
  health: {
    backend: 'online' | 'offline';
    database: 'online' | 'offline';
    ollama: 'ready' | 'loading' | 'error' | 'not_loaded';
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

#### Profile & API Key Configuration Flow
1. User clicks "Settings" in profile dropdown menu in left sidebar
2. Navigates to `/profile` page
3. Scrolls to API Keys Management section
4. Click "Add Key" or "Edit" for specific provider
5. Modal opens with secure input field
6. Key entered and validated via test connection
7. If valid, key encrypted and stored via `POST /api/profile/api-keys`
8. Models requiring that provider become available in model selector

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

## 7 — Backend Architecture & Implementation

### Backend Architecture Overview

The backend follows a **layered service architecture** built on Node.js with Express, designed for scalability, maintainability, and integration with multiple LLM providers. The architecture emphasizes clean separation of concerns, robust error handling, and real-time communication capabilities.

#### Core Architecture Principles
- **Service-oriented design**: Each major feature area is encapsulated in dedicated service classes
- **Provider abstraction**: Unified interface for different LLM providers (Claude, OpenAI, Google AI, Ollama)
- **Streaming-first**: Built-in support for real-time streaming responses via Server-Sent Events
- **Security by design**: End-to-end encryption for API keys, JWT-based authentication, comprehensive input validation
- **Observability**: Structured logging, health checks, and metrics collection at every layer

#### Request Processing Pipeline
```
Client Request → Auth Middleware → Validation → Service Layer → Provider Layer → Database Layer
                     ↓              ↓             ↓              ↓              ↓
               JWT Verification → Schema Check → Business Logic → Model Call → Data Persistence
                     ↓              ↓             ↓              ↓              ↓
               Rate Limiting → Error Handling → Response Format → Streaming → Audit Log
```

### Core Services & Components

#### Authentication Service (`AuthService`)
- **JWT Management**: Token generation, validation, and refresh logic
- **Session Handling**: httpOnly cookie management with secure flags
- **Password Security**: bcrypt hashing with configurable salt rounds
- **Rate Limiting**: Login attempt throttling per IP and user

#### Thread Management Service (`ThreadService`)
- **CRUD Operations**: Create, read, update, delete threads with proper authorization
- **Search Functionality**: Full-text search across thread titles and summaries
- **Pagination**: Cursor-based pagination for efficient large dataset handling
- **Model Context Tracking**: Separate conversation contexts per model within threads

#### Message Processing Service (`MessageService`)
- **Message Storage**: Efficient storage and retrieval with indexing
- **Context Assembly**: Builds conversation context from system prompts, summaries, and recent messages
- **Token Management**: Tracks and limits context tokens per model requirements
- **Streaming Coordination**: Manages real-time message delivery via SSE

#### Model Proxy Service (`ModelProxyService`)
- **Provider Abstraction**: Unified interface for all LLM providers
- **Request Routing**: Intelligently routes requests to appropriate providers
- **Response Normalization**: Standardizes responses across different provider formats
- **Error Handling**: Provider-specific error translation and retry logic

#### Summarization Service (`SummarizationService`)
- **Async Processing**: Non-blocking summarization triggered per conversation
- **Context-Aware**: Uses conversation history to generate relevant summaries
- **Token Optimization**: Maintains summaries within 300-700 token range
- **Fallback Handling**: Graceful degradation when summarization fails

#### Health Monitoring Service (`HealthService`)
- **System Status**: Real-time monitoring of all system components
- **Database Connectivity**: Continuous database health checks
- **Model Availability**: Tracks loading status of local and remote models
- **Performance Metrics**: Response times, error rates, and resource utilization

### Authentication & Authorization

#### JWT Implementation
- **Token Structure**:
  ```json
  {
    "sub": "user_id",
    "username": "user@example.com",
    "iat": 1234567890,
    "exp": 1234567890,
    "scope": ["read", "write"]
  }
  ```
- **Security Features**:
  - HS256 signing with rotating secrets
  - Short-lived tokens (15 minutes) with refresh capability
  - httpOnly cookies with secure and sameSite flags
  - Automatic token refresh before expiration

#### Authorization Middleware
- **Route Protection**: Middleware validates JWT on protected endpoints
- **Role-Based Access**: Future-ready for admin/user role separation
- **API Key Validation**: Verifies user has required API keys for model access
- **Rate Limiting**: Per-user request throttling to prevent abuse

### Model Integration Layer

#### Unified Model Interface
```typescript
interface ModelProvider {
  name: string;
  authenticate(apiKey: string): Promise<boolean>;
  generateCompletion(request: CompletionRequest): Promise<CompletionResponse>;
  generateStream(request: CompletionRequest): AsyncGenerator<StreamChunk>;
  getStatus(): Promise<ModelStatus>;
  loadModel?(modelId: string): Promise<LoadResult>;
}
```

#### Provider Implementations
- **Claude Provider**: Anthropic API integration with streaming support
- **OpenAI Provider**: GPT model access with function calling capabilities
- **Google AI Provider**: Gemini model integration
- **Ollama Provider**: Local model management with loading orchestration

#### Model Loading Orchestration
- **On-Demand Loading**: Local models loaded only when requested
- **Status Tracking**: Real-time loading progress via SSE or polling
- **Resource Management**: Memory and CPU optimization for model operations
- **Failure Recovery**: Automatic retry logic with exponential backoff

### Real-time Communication

#### Server-Sent Events (SSE) Implementation
- **Connection Management**: Maintains persistent connections for streaming
- **Message Formatting**: Structured SSE messages with proper event types
- **Error Recovery**: Automatic reconnection with exponential backoff
- **Heartbeat**: Keep-alive messages to prevent connection timeouts

#### Streaming Architecture
```typescript
interface StreamingResponse {
  eventType: 'token' | 'complete' | 'error' | 'status';
  data: {
    content?: string;
    messageId?: string;
    status?: string;
    error?: ErrorDetails;
  };
  timestamp: string;
}
```

### Performance Optimization

#### Caching Strategy (Redis-free Initial Implementation)
- **In-Memory Caching**: LRU cache for frequently accessed data
- **Model Status Caching**: Cache model availability to reduce API calls
- **Thread Metadata Caching**: Cache thread summaries and metadata
- **API Response Caching**: Cache non-sensitive API responses

#### Connection Management
- **HTTP Keep-Alive**: Reuse connections to external APIs
- **Connection Pooling**: Efficient database connection management
- **Request Batching**: Batch non-critical operations where possible

### Security Implementation

#### API Key Management
- **Encryption at Rest**: AES-256-GCM encryption with KMS-managed keys
- **Secure Transmission**: TLS 1.3 for all external communications
- **Key Rotation**: Support for API key updates without service interruption
- **Access Logging**: Audit trail for all API key usage

#### Input Validation & Sanitization
- **Schema Validation**: Joi/Zod schemas for all request payloads
- **Content Filtering**: Sanitize user inputs to prevent injection attacks
- **File Upload Security**: Validate file types and scan for malicious content
- **Rate Limiting**: Configurable limits per user and endpoint

### Error Handling & Logging

#### Structured Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": {
      "field": "temperature",
      "expected": "number between 0 and 1"
    },
    "requestId": "req_123456789",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

#### Logging Strategy
- **Structured Logging**: JSON format with correlation IDs
- **Log Levels**: DEBUG, INFO, WARN, ERROR with appropriate filtering
- **Sensitive Data Redaction**: Automatic redaction of API keys and personal data
- **Request Tracing**: Full request lifecycle tracking for debugging

---

### API Endpoints (Comprehensive Specification)

All API endpoints use RESTful conventions with JSON request/response bodies. Authentication is handled via JWT tokens in httpOnly cookies. Rate limiting is applied per user and endpoint.

#### Standard Response Format

**Success Response:**
```json
{
  "success": true,
  "data": { /* endpoint-specific data */ },
  "pagination": { /* for list endpoints */ },
  "timestamp": "2024-01-15T10:30:00Z",
  "requestId": "req_123456789"
}
```

**Error Response:**
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": { /* optional error details */ }
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "requestId": "req_123456789"
}
```

**Pagination Format (for list endpoints):**
```json
{
  "pagination": {
    "cursor": "base64_encoded_cursor",
    "limit": 20,
    "hasMore": true,
    "totalCount": 150
  }
}
```

#### Authentication Endpoints

##### POST /api/login
**Description**: Authenticate user and create session
**Authentication**: Public
**Rate Limit**: 5 requests per minute per IP

**Request:**
```json
{
  "username": "user@example.com",
  "password": "securepassword123"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user_123",
      "username": "user@example.com",
      "displayName": "John Doe",
      "createdAt": "2024-01-01T00:00:00Z"
    },
    "expiresIn": 900
  }
}
```
**Note**: JWT token set as httpOnly, secure, sameSite cookie

**Error Responses:**
- `401 INVALID_CREDENTIALS`: Invalid username or password
- `429 RATE_LIMITED`: Too many login attempts
- `400 VALIDATION_ERROR`: Missing or invalid request fields

##### POST /api/logout
**Description**: Clear authentication session
**Authentication**: Optional
**Rate Limit**: None

**Request Body**: Empty

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "message": "Successfully logged out"
  }
}
```

##### GET /api/health
**Description**: System health check
**Authentication**: Public
**Rate Limit**: 10 requests per minute per IP

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "uptime": 86400,
    "services": {
      "database": {
        "status": "healthy",
        "provider": "firebase",
        "latency": 45
      },
      "ollama": {
        "status": "loaded",
        "model": "gemma-2b",
        "memory": "2.1GB"
      },
      "externalApis": {
        "claude": "reachable",
        "openai": "reachable",
        "google": "reachable"
      }
    },
    "version": "1.2.3",
    "environment": "production"
  }
}
```

##### GET /api/version
**Description**: Get application version information
**Authentication**: Required
**Rate Limit**: 30 requests per minute per user

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "version": "1.2.3",
    "buildTime": "2024-01-15T08:00:00Z",
    "commit": "abc123def456",
    "environment": "production"
  }
}
```

#### Profile Management Endpoints

##### GET /api/profile
**Description**: Get user profile information
**Authentication**: Required
**Rate Limit**: 60 requests per minute per user

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user_123",
      "username": "user@example.com",
      "displayName": "John Doe",
      "createdAt": "2024-01-01T00:00:00Z",
      "settings": {
        "defaultModel": "claude-opus",
        "defaultTemperature": 0.7,
        "systemPrompt": "You are a helpful assistant..."
      },
      "apiKeys": {
        "claude": {
          "configured": true,
          "lastUsed": "2024-01-14T10:30:00Z"
        },
        "openai": {
          "configured": false
        },
        "google": {
          "configured": true,
          "lastUsed": "2024-01-13T15:20:00Z"
        }
      }
    }
  }
}
```

##### PATCH /api/profile
**Description**: Update user profile settings
**Authentication**: Required
**Rate Limit**: 20 requests per minute per user

**Request:**
```json
{
  "displayName": "John Smith",
  "settings": {
    "defaultModel": "gpt-4",
    "defaultTemperature": 0.8,
    "systemPrompt": "You are an expert coding assistant..."
  }
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user_123",
      "username": "user@example.com",
      "displayName": "John Smith",
      "settings": {
        "defaultModel": "gpt-4",
        "defaultTemperature": 0.8,
        "systemPrompt": "You are an expert coding assistant..."
      }
    }
  }
}
```

##### POST /api/profile/api-keys
**Description**: Add or update API keys for external providers
**Authentication**: Required
**Rate Limit**: 5 requests per minute per user

**Request:**
```json
{
  "claude": "sk-ant-api03-...",
  "openai": "sk-proj-...",
  "google": "AIza..."
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "updated": ["claude", "openai"],
    "verified": {
      "claude": true,
      "openai": true,
      "google": false
    }
  }
}
```

#### Model Management Endpoints

##### GET /api/models
**Description**: Get available model catalog
**Authentication**: Required
**Rate Limit**: 60 requests per minute per user

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "models": [
      {
        "id": "claude-opus",
        "name": "Claude 3 Opus",
        "provider": "anthropic",
        "apiKeyRequired": "claude",
        "capabilities": ["text", "reasoning", "analysis"],
        "contextLength": 200000,
        "pricing": {
          "input": 0.015,
          "output": 0.075
        },
        "temperatureRange": {
          "min": 0,
          "max": 1,
          "step": 0.1,
          "default": 0.7
        }
      },
      {
        "id": "gemma-2b-local",
        "name": "Gemma 2B (Local)",
        "provider": "ollama",
        "apiKeyRequired": null,
        "requiresLoading": true,
        "capabilities": ["text", "basic-reasoning"],
        "contextLength": 8192,
        "pricing": null
      }
    ]
  }
}
```

##### GET /api/models/:modelId/status
**Description**: Get specific model status and loading information
**Authentication**: Required
**Rate Limit**: 30 requests per minute per user

**URL Parameters:**
- `modelId`: Model identifier (e.g., "gemma-2b-local")

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "modelId": "gemma-2b-local",
    "status": "loaded",
    "loadProgress": 100,
    "memoryUsage": "2.1GB",
    "lastLoaded": "2024-01-15T09:30:00Z",
    "estimatedLoadTime": null
  }
}
```

**Status Values:**
- `not_loaded`: Model not currently loaded
- `loading`: Model currently loading (includes progress)
- `loaded`: Model ready for use
- `error`: Loading failed
- `unavailable`: Model not available on this instance

##### POST /api/models/:modelId/load
**Description**: Load a local model (Ollama)
**Authentication**: Required
**Rate Limit**: 3 requests per minute per user

**URL Parameters:**
- `modelId`: Model identifier (e.g., "gemma-2b-local")

**Request Body**: Empty

**Success Response (202):**
```json
{
  "success": true,
  "data": {
    "modelId": "gemma-2b-local",
    "jobId": "load_job_123",
    "status": "loading",
    "estimatedTime": 180
  }
}
```

**SSE Stream URL**: `/api/models/:modelId/load/stream?jobId=load_job_123`

#### Thread Management Endpoints

##### GET /api/threads
**Description**: List user's threads with pagination and search
**Authentication**: Required
**Rate Limit**: 100 requests per minute per user

**Query Parameters:**
- `limit` (optional): Number of items per page (default: 20, max: 100)
- `cursor` (optional): Pagination cursor for next page
- `q` (optional): Search query (searches title and summary)
- `orderBy` (optional): Sort order ("created_at", "updated_at") default: "updated_at"
- `order` (optional): Sort direction ("asc", "desc") default: "desc"

**Example Request:**
```
GET /api/threads?limit=20&q=architecture&orderBy=updated_at&order=desc
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "threads": [
      {
        "id": "thread_123",
        "title": "System Architecture Discussion",
        "createdAt": "2024-01-14T10:00:00Z",
        "updatedAt": "2024-01-15T10:30:00Z",
        "messageCount": 15,
        "activeModels": ["claude-opus", "gpt-4"],
        "lastMessage": {
          "content": "That's a great approach for microservices...",
          "role": "assistant",
          "model": "claude-opus",
          "timestamp": "2024-01-15T10:30:00Z"
        },
        "summary": "Discussion about microservice architecture patterns and best practices"
      }
    ]
  },
  "pagination": {
    "cursor": "eyJjcmVhdGVkQXQiOiIyMDI0LTAxLTE1VDEwOjMwOjAwWiIsImlkIjoidGhyZWFkXzEyMyJ9",
    "limit": 20,
    "hasMore": true,
    "totalCount": 157
  }
}
```

##### POST /api/threads
**Description**: Create a new thread
**Authentication**: Required
**Rate Limit**: 10 requests per minute per user

**Request:**
```json
{
  "title": "New Discussion Topic"
}
```

**Success Response (201):**
```json
{
  "success": true,
  "data": {
    "thread": {
      "id": "thread_456",
      "title": "New Discussion Topic",
      "createdAt": "2024-01-15T10:45:00Z",
      "updatedAt": "2024-01-15T10:45:00Z",
      "messageCount": 0,
      "activeModels": [],
      "summary": null
    }
  }
}
```

##### GET /api/threads/:threadId
**Description**: Get thread metadata and summary
**Authentication**: Required (thread owner only)
**Rate Limit**: 60 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "thread": {
      "id": "thread_123",
      "title": "System Architecture Discussion",
      "createdAt": "2024-01-14T10:00:00Z",
      "updatedAt": "2024-01-15T10:30:00Z",
      "messageCount": 15,
      "models": {
        "claude-opus": {
          "messageCount": 8,
          "lastMessageAt": "2024-01-15T10:30:00Z",
          "summary": "Discussion focused on microservice patterns...",
          "summaryUpdatedAt": "2024-01-15T10:25:00Z"
        },
        "gpt-4": {
          "messageCount": 7,
          "lastMessageAt": "2024-01-15T09:45:00Z",
          "summary": "Comparison of architectural approaches...",
          "summaryUpdatedAt": "2024-01-15T09:40:00Z"
        }
      }
    }
  }
}
```

##### DELETE /api/threads/:threadId
**Description**: Delete a thread and all its messages
**Authentication**: Required (thread owner only)
**Rate Limit**: 5 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "deletedThread": "thread_123",
    "deletedMessages": 15
  }
}
```

##### PATCH /api/threads/:threadId/title
**Description**: Update thread title
**Authentication**: Required (thread owner only)
**Rate Limit**: 20 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier

**Request:**
```json
{
  "title": "Updated Thread Title"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "thread": {
      "id": "thread_123",
      "title": "Updated Thread Title",
      "updatedAt": "2024-01-15T10:50:00Z"
    }
  }
}
```

#### Message Management Endpoints

##### GET /api/threads/:threadId/models/:modelId/messages
**Description**: Get messages for specific model in thread
**Authentication**: Required (thread owner only)
**Rate Limit**: 100 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier
- `modelId`: Model identifier

**Query Parameters:**
- `limit` (optional): Number of messages (default: 50, max: 200)
- `cursor` (optional): Pagination cursor
- `order` (optional): "asc" or "desc" (default: "desc")

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "messages": [
      {
        "id": "msg_123",
        "threadId": "thread_123",
        "modelId": "claude-opus",
        "role": "user",
        "content": "What are the best practices for microservices?",
        "tokens": 12,
        "createdAt": "2024-01-15T10:20:00Z",
        "status": "complete"
      },
      {
        "id": "msg_124",
        "threadId": "thread_123",
        "modelId": "claude-opus",
        "role": "assistant",
        "content": "Here are the key microservice patterns and best practices...",
        "tokens": 156,
        "temperature": 0.7,
        "createdAt": "2024-01-15T10:21:00Z",
        "status": "complete"
      }
    ]
  },
  "pagination": {
    "cursor": "eyJjcmVhdGVkQXQiOiIyMDI0LTAxLTE1VDEwOjIxOjAwWiIsImlkIjoibXNnXzEyNCJ9",
    "limit": 50,
    "hasMore": false
  }
}
```

##### POST /api/threads/:threadId/models/:modelId/messages
**Description**: Send message to model and get response
**Authentication**: Required (thread owner only)
**Rate Limit**: 20 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier
- `modelId`: Model identifier

**Request:**
```json
{
  "content": "Explain the benefits of event-driven architecture",
  "temperature": 0.8,
  "stream": true
}
```

**Non-streaming Response (200):**
```json
{
  "success": true,
  "data": {
    "userMessage": {
      "id": "msg_125",
      "content": "Explain the benefits of event-driven architecture",
      "createdAt": "2024-01-15T10:35:00Z"
    },
    "assistantMessage": {
      "id": "msg_126",
      "content": "Event-driven architecture offers several key benefits...",
      "tokens": 234,
      "temperature": 0.8,
      "createdAt": "2024-01-15T10:35:30Z"
    }
  }
}
```

**Streaming Response (SSE):**
When `stream: true`, the response is sent via Server-Sent Events:

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: message_start
data: {"messageId": "msg_126", "timestamp": "2024-01-15T10:35:00Z"}

event: token
data: {"token": "Event", "messageId": "msg_126"}

event: token
data: {"token": "-driven", "messageId": "msg_126"}

event: message_complete
data: {"messageId": "msg_126", "totalTokens": 234, "timestamp": "2024-01-15T10:35:30Z"}

event: close
data: {}
```

##### POST /api/threads/:threadId/models/:modelId/summarize
**Description**: Manually trigger summarization for thread/model
**Authentication**: Required (thread owner only)
**Rate Limit**: 5 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier
- `modelId`: Model identifier

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "summary": "This conversation explored event-driven architecture patterns, discussing benefits like loose coupling, scalability, and resilience. Key topics included message queues, event sourcing, and CQRS patterns.",
    "tokens": 45,
    "generatedAt": "2024-01-15T10:40:00Z",
    "modelUsed": "claude-opus"
  }
}
```

#### Error Handling

All endpoints return appropriate HTTP status codes:

- `200 OK`: Successful request
- `201 Created`: Resource created successfully
- `202 Accepted`: Request accepted for async processing
- `400 Bad Request`: Invalid request format or parameters
- `401 Unauthorized`: Authentication required or invalid
- `403 Forbidden`: Access denied to resource
- `404 Not Found`: Resource not found
- `409 Conflict`: Resource conflict (e.g., duplicate creation)
- `422 Unprocessable Entity`: Valid format but business logic error
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error
- `503 Service Unavailable`: Temporary service unavailability

#### Rate Limiting Headers

All responses include rate limiting headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642234567
X-RateLimit-Window: 60
```

---

## 8 — Data Model & Persistence

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