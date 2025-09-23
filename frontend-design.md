# Frontend Implementation Specifications

The frontend follows a **component-driven architecture** using React with TypeScript and Material-UI, organized in layers.

## 1. Application Structure

The frontend uses a structured approach to organization and component hierarchy.

### Directory Layout

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

### State Management Strategy

- **Authentication State**: JWT token, user session, login status
- **Thread Management**: Active thread, thread list, pagination cursors
- **Message Handling**: Streaming messages, conversation history, model contexts
- **Model Configuration**: Available models, loading states, temperature settings
- **UI State**: Sidebar collapse, modal states, notification toasts
- **Health Monitoring**: System status, version info, real-time health checks

## 2. Navigation Structure & Routes

The application uses **React Router** with protected routes and role-based access:

```typescript
/                          → Dashboard (authenticated users only)
/login                     → Login Page (unauthenticated only)
/threads/:threadId         → Thread View (authenticated)
/profile                   → Profile Settings (authenticated)
/404                       → Not Found Page
```

## 3. Detailed Page Specifications

The frontend consists of 5 main pages, focusing on core functionality with a streamlined interface.

### Login Page (`/login`)

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

#### Components
- `LoginForm`: Username/password inputs with validation
- `HealthIndicator`: Shows backend connectivity status
- `VersionDisplay`: Frontend and backend version info
- `ErrorAlert`: Authentication error messages

#### Functionality
- Form validation (client-side)
- JWT token handling via httpOnly cookies
- Redirect to dashboard on successful auth
- Health check display for system status
- Responsive design for mobile/tablet

### Dashboard/Main Chat Interface (`/`)

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

#### Main Components
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

#### Left Sidebar Details
- Thread cards show title, creation time, active models (colored dots)
- Search functionality filters by title and summary
- Infinite scroll with cursor-based pagination
- "New Thread" button creates thread and redirects
- System health status (Backend/Database/Ollama) with green/red indicators
- Profile dropdown menu with "Settings" and "Logout" options
- Version information display (Frontend and Backend versions)
- Collapsible on mobile/tablet

#### Chat Area Details
- Model selector populated from `/api/models`
- Temperature control adapts to model configuration (slider vs toggle)
- Load button appears for local models when not loaded
- Message timestamps and status indicators
- Copy message buttons
- Markdown rendering for assistant responses

### Thread View (`/threads/:threadId`)

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

#### Components
- `ThreadHeader`: Navigation, title editing, sharing options
- `ModelTabs`: Switch between model conversations in thread
- `SummaryPanel`: Collapsible summary display per model
- `MessageTimeline`: Chronological message display with model context
- `MessageActions`: Copy, retry, continue generation
- `ModelContextSwitcher`: Change active model for new messages

#### Functionality
- Each model maintains separate conversation context within thread
- Summaries update asynchronously and display under each model tab
- Message history shows which model each message was sent to
- Model switching preserves input text
- Infinite scroll for message history
- Title editing with auto-save

### Profile Settings (`/profile`)

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

#### Components
- `UserInfoCard`: Display name editing, read-only username
- `DefaultSettingsCard`: Model and temperature preferences
- `SystemPromptEditor`: Large text area with character counting
- `APIKeyManager`: Masked key display with configuration status

### 404 Not Found (`/404` or unmatched routes)

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

## 4. Component Library Specifications

The application uses Material-UI as the foundational component library with custom components built on top.

### Material-UI Based Components

#### Theme Configuration
- **Primary Color**: Clear blue theme for brand consistency
- **Typography**: Material-UI typography scale with clear, readable fonts
- **Spacing**: Consistent 8px grid system
- **Color Palette**: Light theme with clear contrast ratios for accessibility

#### Navigation Components
- `AppBar` (MUI): Simple header with app name and sidebar toggle
- `LeftSidebar`: Custom collapsible sidebar with MUI components
- `Drawer` (MUI): Mobile sidebar implementation

#### Data Display Components
- `ThreadCard`: Custom card using MUI Card component
- `MessageBubble`: Custom component with MUI Paper and Typography
- `Chip` (MUI): Status indicators for health/loading states
- `Badge` (MUI): Version and notification indicators

#### Input Components
- `Autocomplete` (MUI): Model selector with search and filtering
- `Slider` (MUI): Temperature control slider
- `TextField` (MUI): Message input and form fields
- `FormControl` (MUI): Structured form inputs

#### Feedback Components
- `CircularProgress` (MUI): Loading indicators
- `LinearProgress` (MUI): Model loading progress bars
- `Snackbar` (MUI): Success/error toast notifications
- `Dialog` (MUI): Modal dialogs and confirmations

#### Layout Components
- `Container` (MUI): Page wrapper with responsive breakpoints
- `Grid` (MUI): Responsive grid system
- `Box` (MUI): Flexible layout container
- `Stack` (MUI): Vertical/horizontal component stacking

## 5. State Management Approach

The application uses Zustand for state management with a structured store design.

### Zustand Store Structure

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
    lastCheck: number | null; // epoch timestamp
    checkHealth: () => Promise<void>;
  };
}
```

## 6. User Interaction Flows

Key user journeys and their implementation patterns.

### Thread Creation & Message Flow

1. User clicks "New Thread" in sidebar
2. User types first message and selects model
3. `createThread()` API call:
   - Stores message data in DB
   - Initiates async LLM call for response
   - Generates 3-word title using local Ollama (synchronous)
   - Returns thread ID, message IDs, and generated title
4. User redirected to `/threads/:threadId` with generated title
5. SSE connection established to stream assistant response
6. Real-time token updates via DB watcher
7. Thread updated when response complete

### Model Loading Flow

1. User selects local model (Gemma) from dropdown
2. `GET /api/models/gemma-2b/status` checks current state
3. If not loaded, "Load Model" button appears
4. User clicks load, `POST /api/models/gemma-2b/load` called
5. Progress bar shows loading status via polling or SSE
6. Model becomes available for conversation

### Profile & API Key Configuration Flow

1. User clicks "Settings" in profile dropdown menu in left sidebar
2. Navigates to `/profile` page
3. Scrolls to API Keys Management section
4. Click "Add Key" or "Edit" for specific provider
5. Modal opens with secure input field
6. Key entered and validated via test connection
7. If valid, key encrypted and stored via `POST /api/profile/api-keys`
8. Models requiring that provider become available in model selector

## 7. Responsive Design Strategy

The application adapts across different screen sizes and devices.

### Breakpoint System

- **Mobile**: < 640px (single column, collapsible sidebar)
- **Tablet**: 640px - 1024px (sidebar overlay, compact controls)
- **Desktop**: > 1024px (full layout with sidebar)

### Mobile Adaptations

- Sidebar becomes slide-over modal
- Message input moves to bottom fixed position
- Model selector becomes bottom sheet
- Chat area uses full viewport height
- Touch-optimized button sizes (44px minimum)

### Tablet Adaptations

- Sidebar width reduces to 280px
- Model controls stack vertically
- Temperature slider becomes larger touch target
- Thread cards use horizontal layout

### Accessibility Features

- ARIA labels for all interactive elements
- Keyboard navigation support
- Focus management for modals and dropdowns
- Screen reader announcements for streaming messages
- High contrast mode support
- Reduced motion preferences

## 8. Performance Optimizations

Strategies to ensure optimal application performance.

### Code Splitting

- Route-based code splitting for each page
- Lazy loading of admin panel components
- Dynamic imports for heavy dependencies

### State Optimization

- Virtualized scrolling for large message lists
- Debounced search input (300ms)
- Memoized expensive computations
- Selective re-renders with React.memo

### Network Optimization

- Request deduplication for model status checks
- Optimistic updates for message sending
- Background prefetching of thread data
- Compression for large message payloads