# Data Model & Persistence Architecture

This document provides comprehensive specifications for the data model, persistence layer, and initialization system for the LLM Chat application.

## Table of Contents

1. [Database Architecture Overview](#1-database-architecture-overview)
2. [Collection Schemas](#2-collection-schemas)
3. [Data Relationships & Integrity](#3-data-relationships--integrity)
4. [Indexing Strategy](#4-indexing-strategy)
5. [Database Initialization Script](#5-database-initialization-script)
6. [Model Configuration Data](#6-model-configuration-data)
7. [Design Principles](#7-design-principles)

---

## 1. Database Architecture Overview

### Technology Stack

The application uses **Firebase Realtime Database** for both production and development environments:
- **Production**: Firebase Realtime Database hosted on Google Cloud
- **Development**: Firebase Emulator Suite running locally

### High-Level Structure

The database follows a document-oriented design optimized for Firebase Realtime Database real-time capabilities:

```
Database Root
├── users/                    # User profiles and encrypted API keys
├── threads/                  # Conversation threads with metadata
├── messages/                 # Individual messages (separate collection)
├── models_config/            # Model catalog and configuration
├── operations/               # Async operation tracking
└── system/                   # Application metadata and version info
```

### Design Principles

- **Denormalization for Performance**: Optimize for read operations and real-time updates
- **Atomic Operations**: Design updates to be atomic within document boundaries
- **Scalable Pagination**: Use cursor-based pagination for all list operations
- **Real-time Capable**: Structure supports Firebase real-time listeners
- **Security First**: All API keys encrypted, content sanitized, access controlled

---

## 2. Collection Schemas

### 2.1 Users Collection

**Purpose**: Store user profiles, authentication data, settings, and encrypted API keys.

```typescript
interface UserDocument {
  // Identity
  id: string;                    // Primary key: "user_123"
  username: string;              // Unique email address
  displayName: string;           // User's display name
  passwordHash: string;          // bcrypt hashed password

  // Timestamps
  createdAt: number;             // Epoch timestamp
  updatedAt: number;             // Last profile update
  lastLoginAt?: number;          // Last successful login

  // Settings
  settings: {
    defaultModel: string;        // Default model ID
    defaultTemperature: number;  // Default temperature (0-1)
    systemPrompt: string;        // User's default system prompt
    streamingEnabled: boolean;   // Streaming preference
    theme?: 'light' | 'dark';    // UI theme preference
  };

  // Encrypted API Keys
  apiKeys: {
    claude?: {
      encryptedKey: string;      // AES-256-GCM encrypted
      keyHash: string;           // SHA-256 hash for verification
      lastUsed?: number;         // Last usage timestamp
      isValid: boolean;          // Last validation result
    };
    openai?: {
      encryptedKey: string;
      keyHash: string;
      lastUsed?: number;
      isValid: boolean;
    };
    google?: {
      encryptedKey: string;
      keyHash: string;
      lastUsed?: number;
      isValid: boolean;
    };
  };

  // Usage tracking
  usage: {
    totalThreads: number;        // Total threads created
    totalMessages: number;       // Total messages sent
    totalTokens: number;         // Total tokens consumed
    lastActiveAt: number;        // Last activity timestamp
  };

  // Account status
  status: 'active' | 'suspended' | 'deleted';
  suspensionReason?: string;

  // Version tracking
  schemaVersion: number;         // For schema evolution
}
```

### 2.2 Threads Collection

**Purpose**: Store conversation threads with model-specific metadata and summaries.

```typescript
interface ThreadDocument {
  // Identity
  id: string;                    // Primary key: "thread_123"
  userId: string;                // Foreign key to users collection

  // Basic metadata
  title: string;                 // Thread title (auto-generated or user-set)
  createdAt: number;             // Creation timestamp
  updatedAt: number;             // Last activity timestamp

  // Model-specific tracking
  models: {
    [modelId: string]: {
      messageCount: number;      // Messages for this model
      lastMessageAt: number;     // Last message timestamp
      summary?: string;          // AI-generated summary (300-700 tokens)
      summaryUpdatedAt?: number; // Summary generation timestamp
      summaryTokens?: number;    // Token count of summary
      lastTemperature?: number;  // Last used temperature
      totalTokensUsed: number;   // Total tokens consumed
      status: 'active' | 'archived' | 'error';
    };
  };

  // Thread statistics
  totalMessages: number;         // Total across all models
  activeModels: string[];        // List of models with messages

  // Thread settings
  settings: {
    isArchived: boolean;         // User archived status
    isPinned: boolean;           // Pinned to top of list
    tags?: string[];             // User-defined tags
    color?: string;              // UI color coding
  };

  // Sharing and collaboration (future)
  sharing?: {
    isPublic: boolean;
    shareToken?: string;
    allowedUsers?: string[];
  };

  // Version tracking
  schemaVersion: number;
}
```

### 2.3 Messages Collection

**Purpose**: Store individual messages with streaming token support and metadata.

```typescript
interface MessageDocument {
  // Identity and relationships
  id: string;                    // Primary key: "msg_123"
  threadId: string;              // Foreign key to threads
  modelId: string;               // Model used for this message
  userId: string;                // Message owner (denormalized)

  // Message content
  role: 'user' | 'assistant' | 'system';
  content?: string;              // For user/system messages (full content)
  tokens?: string[];             // For assistant messages (token array)

  // Message status
  status: 'pending' | 'generating' | 'complete' | 'failed' | 'cancelled';
  isStreaming: boolean;          // Currently receiving tokens

  // Generation parameters
  temperature?: number;          // Used temperature (assistant messages)
  systemPrompt?: string;         // System prompt used (denormalized)
  contextTokens?: number;        // Context tokens sent to model

  // Token and timing information
  totalTokens?: number;          // Total tokens in complete message
  generationStartedAt?: number;  // When LLM call initiated
  generationCompletedAt?: number; // When response fully received
  estimatedCompletionTime?: number; // ETA during generation

  // Error handling
  error?: {
    code: string;                // Error code (TIMEOUT, API_ERROR, etc.)
    message: string;             // Human-readable error
    details?: any;               // Additional error details
    timestamp: number;           // Error occurrence time
    retryCount: number;          // Number of retry attempts
  };

  // Metadata
  createdAt: number;             // Message creation time
  updatedAt: number;             // Last update time

  // Message quality and feedback
  feedback?: {
    rating: 1 | 2 | 3 | 4 | 5;   // User rating
    helpful: boolean;            // Helpful flag
    comments?: string;           // User feedback
  };

  // Version tracking
  schemaVersion: number;
}
```

### 2.4 Models Config Collection

**Purpose**: Centralized model catalog with capabilities and configuration.

```typescript
interface ModelConfigDocument {
  // Model identity
  id: string;                    // Primary key: "claude-opus-4", "gemma-2b-local"
  name: string;                  // Display name: "Claude 4 Opus"
  provider: 'anthropic' | 'openai' | 'google' | 'ollama';

  // Authentication requirements
  apiKeyType?: 'claude' | 'openai' | 'google' | null; // null for local models
  requiresApiKey: boolean;

  // Model capabilities
  capabilities: {
    text: boolean;               // Text generation
    reasoning: boolean;          // Complex reasoning
    coding: boolean;             // Code generation
    analysis: boolean;           // Data analysis
    vision?: boolean;            // Image understanding (future)
    function_calling?: boolean;  // Function/tool calling
  };

  // Technical specifications
  specs: {
    contextLength: number;       // Maximum context tokens
    maxOutputTokens?: number;    // Maximum output tokens
    inputTokenCost?: number;     // Cost per 1K input tokens (USD)
    outputTokenCost?: number;    // Cost per 1K output tokens (USD)
    currency: 'USD';             // Cost currency

    // Local model specific
    modelSize?: string;          // "2B", "7B", "13B"
    quantization?: string;       // "Q4_0", "Q8_0", etc.
    memoryRequirement?: number;  // GB of RAM required
    diskSpace?: number;          // GB of disk space
    loadTimeEstimate?: number;   // Seconds to load
  };

  // UI configuration
  ui: {
    temperatureType: 'range' | 'enum';
    temperatureOptions: {
      min?: number;              // For range type
      max?: number;              // For range type
      step?: number;             // For range type
      values?: number[];         // For enum type
      default: number;           // Default value
    };
    displayOrder: number;        // Sort order in UI
    color?: string;              // Brand color
    icon?: string;               // Model icon/logo
    category: 'cloud' | 'local' | 'experimental';
  };

  // Operational status
  status: {
    isAvailable: boolean;        // Currently available
    requiresLoading: boolean;    // Local model requiring load
    loadingSupported: boolean;   // Can be loaded on demand

    // Health monitoring
    lastHealthCheck?: number;    // Last API health check
    healthStatus: 'healthy' | 'degraded' | 'down' | 'unknown';
    responseTimeMs?: number;     // Average response time
    errorRate?: number;          // Error rate percentage
  };

  // Feature flags and limits
  features: {
    streamingSupported: boolean;
    summaryGeneration: boolean;
    titleGeneration: boolean;
    batchProcessing?: boolean;

    // Rate limits
    requestsPerMinute?: number;
    requestsPerDay?: number;
    tokensPerMinute?: number;
  };

  // Version and updates
  modelVersion?: string;         // Model version identifier
  lastUpdated: number;           // Configuration last updated
  deprecatedAt?: number;         // Deprecation timestamp
  removedAt?: number;            // Removal timestamp

  // Version tracking
  schemaVersion: number;
}
```

### 2.5 Operations Collection

**Purpose**: Track async operations like summarization and title generation.

```typescript
interface OperationDocument {
  // Identity
  id: string;                    // Primary key: "op_123"
  type: 'summarize' | 'generate_title' | 'load_model' | 'migrate_data';

  // Relationships
  userId: string;                // Operation owner
  threadId?: string;             // Related thread (if applicable)
  modelId?: string;              // Related model (if applicable)
  messageId?: string;            // Related message (if applicable)

  // Operation status
  status: 'pending' | 'running' | 'completed' | 'failed' | 'cancelled';
  progress: number;              // Percentage complete (0-100)

  // Execution details
  startedAt: number;             // Operation start time
  completedAt?: number;          // Operation completion time
  timeoutAt: number;             // Operation timeout deadline

  // Operation parameters
  parameters: {
    [key: string]: any;          // Operation-specific parameters
  };

  // Results
  result?: {
    data?: any;                  // Operation result data
    summary?: string;            // For summarization operations
    title?: string;              // For title generation
    tokensUsed?: number;         // Tokens consumed
    modelUsed?: string;          // Model used for operation
  };

  // Error handling
  error?: {
    code: string;
    message: string;
    details?: any;
    retryCount: number;
    nextRetryAt?: number;
  };

  // Metadata
  createdAt: number;
  updatedAt: number;

  // Version tracking
  schemaVersion: number;
}
```

### 2.6 System Collection

**Purpose**: Application metadata, configuration, and operational data.

```typescript
interface SystemDocument {
  // Document type identifier
  type: 'version' | 'config' | 'stats' | 'health';

  // Version information
  version?: {
    application: string;         // Application version
    schema: number;              // Current schema version
    buildTime: number;           // Build timestamp
    gitCommit: string;           // Git commit hash
    environment: 'development' | 'staging' | 'production';
  };

  // Global configuration
  config?: {
    features: {
      userRegistration: boolean;
      modelLoading: boolean;
      streaming: boolean;
      summarization: boolean;
    };

    limits: {
      maxThreadsPerUser: number;
      maxMessagesPerThread: number;
      maxTokensPerMessage: number;
      maxConcurrentConnections: number;
      rateLimit: {
        requestsPerMinute: number;
        requestsPerHour: number;
      };
    };

    defaults: {
      temperature: number;
      model: string;
      systemPrompt: string;
    };
  };

  // Application statistics
  stats?: {
    totalUsers: number;
    totalThreads: number;
    totalMessages: number;
    totalTokensProcessed: number;

    dailyStats: {
      date: string;              // YYYY-MM-DD format
      newUsers: number;
      newThreads: number;
      messagesGenerated: number;
      tokensProcessed: number;
      apiCalls: number;
    }[];

    modelUsage: {
      [modelId: string]: {
        totalCalls: number;
        totalTokens: number;
        avgResponseTime: number;
        errorRate: number;
      };
    };
  };

  // Health monitoring
  health?: {
    lastUpdateAt: number;
    services: {
      database: {
        status: 'healthy' | 'degraded' | 'down';
        latency: number;
        connectionCount: number;
        lastError?: string;
      };

      ollama: {
        status: 'ready' | 'loading' | 'error' | 'unavailable';
        loadedModels: string[];
        memoryUsage: number;
        lastError?: string;
      };

      externalApis: {
        [provider: string]: {
          status: 'reachable' | 'unreachable';
          latency: number;
          lastCheck: number;
          errorCount: number;
        };
      };
    };
  };

  // Timestamps
  createdAt: number;
  updatedAt: number;

  // Version tracking
  schemaVersion: number;
}
```

---

## 3. Data Relationships & Integrity

### 3.1 Primary Relationships

```
Users (1) ────────── (N) Threads
   │                     │
   │                     │
   └─ (1) ────────── (N) Messages
   │                     │
   │                     │
   └─ (1) ────────── (N) Operations

Models_Config (1) ── (N) Messages (via modelId)
Threads (1) ─────── (N) Messages (via threadId)
```

### 3.2 Referential Integrity Rules

#### User Deletion
- **CASCADE**: Delete all user's threads, messages, and operations
- **PRESERVE**: Maintain anonymized statistics in system collection
- **ENCRYPT**: Overwrite API keys with random data before deletion

#### Thread Deletion
- **CASCADE**: Delete all messages within the thread
- **UPDATE**: Remove thread references from operations
- **MAINTAIN**: Keep operation records for audit purposes

#### Model Configuration Changes
- **SOFT DELETE**: Mark as deprecated rather than hard delete
- **MAINTAIN**: Preserve message history even for deprecated models
- **MIGRATE**: Update references in active threads and user settings

### 3.3 Data Consistency Patterns

#### Atomic Operations
- Message token streaming uses atomic array append operations
- Thread statistics updated atomically with message creation
- User usage counters updated atomically with API calls
- Firebase transaction functions ensure consistency

---

## 4. Indexing Strategy

### 4.1 Firebase Realtime Database Indexes

#### Primary Access Patterns

```javascript
{
  "rules": {
    // User lookup by username (unique constraint)
    "users": {
      ".indexOn": ["username", "status", "lastLoginAt"]
    },

    // Thread queries by user with pagination
    "threads": {
      ".indexOn": ["userId", "updatedAt", "createdAt", "settings.isArchived"]
    },

    // Message queries by thread and model
    "messages": {
      ".indexOn": ["threadId", "modelId", "createdAt", "status", "userId"]
    },

    // Model config queries
    "models_config": {
      ".indexOn": ["provider", "status.isAvailable", "ui.displayOrder"]
    },

    // Operations tracking
    "operations": {
      ".indexOn": ["userId", "status", "type", "createdAt"]
    },

    // System data queries
    "system": {
      ".indexOn": ["type", "updatedAt"]
    }
  }
}
```

### 4.2 Query Optimization Patterns

#### Common Query Examples

**Thread List with Pagination:**
```javascript
// Firebase query for user's threads
database.ref('threads')
  .orderByChild('userId')
  .equalTo(userId)
  .orderByChild('updatedAt')
  .limitToLast(20)
```

**Message History for Thread/Model:**
```javascript
// Firebase query for thread messages
database.ref('messages')
  .orderByChild('threadId')
  .equalTo(threadId)
  .orderByChild('modelId')
  .equalTo(modelId)
  .orderByChild('createdAt')
  .limitToLast(50)
```

**Real-time Message Streaming:**
```javascript
// Firebase listener for new tokens
database.ref(`messages/${messageId}/tokens`)
  .on('value', (snapshot) => {
    // Handle token updates
  });
```

---

## 5. Database Initialization Script

### 5.1 Initialization Overview

The database initialization script sets up the basic structure and seed data for a new LLM Chat application instance. This is a one-time setup process that:

- Creates the required collections/nodes
- Sets up initial system configuration
- Populates the models catalog
- Creates necessary indexes
- Sets up security rules

### 5.2 Initialization Process

#### Step 1: Firebase Project Setup
1. Create Firebase project
2. Enable Realtime Database
3. Configure authentication providers
4. Set up security rules

#### Step 2: Collection Initialization
- Initialize empty collections: `users`, `threads`, `messages`, `operations`
- Create system configuration documents
- Populate models_config with supported models

#### Step 3: System Configuration
```javascript
// System version document
{
  type: 'version',
  version: {
    application: '1.0.0',
    schema: 1,
    buildTime: Date.now(),
    gitCommit: process.env.GIT_COMMIT,
    environment: process.env.NODE_ENV
  },
  schemaVersion: 1
}

// System configuration document
{
  type: 'config',
  config: {
    features: {
      userRegistration: true,
      modelLoading: true,
      streaming: true,
      summarization: true
    },
    limits: {
      maxThreadsPerUser: 100,
      maxMessagesPerThread: 1000,
      maxTokensPerMessage: 100000,
      maxConcurrentConnections: 50,
      rateLimit: {
        requestsPerMinute: 60,
        requestsPerHour: 1000
      }
    },
    defaults: {
      temperature: 0.7,
      model: 'claude-sonnet-3.5',
      systemPrompt: 'You are a helpful AI assistant.'
    }
  },
  schemaVersion: 1
}
```

### 5.3 Security Rules Setup

#### Firebase Security Rules
```javascript
{
  "rules": {
    "users": {
      "$userId": {
        ".read": "$userId === auth.uid",
        ".write": "$userId === auth.uid",
        ".validate": "newData.hasChildren(['username', 'displayName', 'settings'])"
      }
    },
    "threads": {
      "$threadId": {
        ".read": "data.child('userId').val() === auth.uid",
        ".write": "data.child('userId').val() === auth.uid",
        ".validate": "newData.hasChildren(['userId', 'title', 'models'])"
      }
    },
    "messages": {
      "$messageId": {
        ".read": "data.child('userId').val() === auth.uid",
        ".write": "data.child('userId').val() === auth.uid"
      }
    },
    "models_config": {
      ".read": "auth != null",
      ".write": false  // Read-only for users
    },
    "operations": {
      "$operationId": {
        ".read": "data.child('userId').val() === auth.uid",
        ".write": "data.child('userId').val() === auth.uid"
      }
    },
    "system": {
      ".read": "auth != null",
      ".write": false  // Read-only for users
    }
  }
}
```

---

## 6. Model Configuration Data

### 6.1 Supported Models

The application supports the following AI models, configured in the `models_config` collection:

#### Ollama (Local Models)
- **Gemma2 2B**: Local inference model for development and privacy-focused usage

#### Claude (Anthropic)
- **Claude 4 Opus**: Latest flagship model for complex reasoning
- **Claude 4 Sonnet**: Balanced performance and speed
- **Claude 3.7 Sonnet**: Enhanced capability version
- **Claude 3.5 Sonnet**: Production-ready general purpose
- **Claude 3.5 Haiku**: Fast, lightweight model

#### Google (Gemini)
- **Gemini 2.5 Pro**: Advanced reasoning and analysis
- **Gemini 2.5 Flash**: Fast response model
- **Gemini 2.5 Flash Light**: Lightweight, efficient model

#### OpenAI
- **GPT-5**: Latest flagship model
- **GPT-5 Mini**: Smaller, faster variant
- **GPT-5 Nano**: Ultra-lightweight model
- **GPT-4.1**: Enhanced version of GPT-4
- **GPT-4.1 Mini**: Compact version of GPT-4.1

### 6.2 Model Configuration Structure

Each model follows the `ModelConfigDocument` interface with specific configurations:

#### Example: Claude 4 Opus Configuration
```javascript
{
  id: 'claude-4-opus',
  name: 'Claude 4 Opus',
  provider: 'anthropic',
  apiKeyType: 'claude',
  requiresApiKey: true,
  capabilities: {
    text: true,
    reasoning: true,
    coding: true,
    analysis: true,
    vision: false,
    function_calling: true
  },
  specs: {
    contextLength: 200000,
    maxOutputTokens: 8192,
    inputTokenCost: 0.015,
    outputTokenCost: 0.075,
    currency: 'USD'
  },
  ui: {
    temperatureType: 'range',
    temperatureOptions: {
      min: 0.0,
      max: 1.0,
      step: 0.1,
      default: 0.7
    },
    displayOrder: 1,
    color: '#FF6B35',
    category: 'cloud'
  },
  status: {
    isAvailable: true,
    requiresLoading: false,
    loadingSupported: false,
    healthStatus: 'unknown'
  },
  features: {
    streamingSupported: true,
    summaryGeneration: true,
    titleGeneration: true,
    requestsPerMinute: 60,
    requestsPerDay: 1000,
    tokensPerMinute: 40000
  },
  schemaVersion: 1
}
```

#### Example: Ollama Gemma2 Configuration
```javascript
{
  id: 'gemma2-2b-local',
  name: 'Gemma2 2B (Local)',
  provider: 'ollama',
  apiKeyType: null,
  requiresApiKey: false,
  capabilities: {
    text: true,
    reasoning: true,
    coding: true,
    analysis: false,
    vision: false,
    function_calling: false
  },
  specs: {
    contextLength: 8192,
    maxOutputTokens: 2048,
    modelSize: '2B',
    quantization: 'Q4_0',
    memoryRequirement: 3,
    diskSpace: 1.6,
    loadTimeEstimate: 10
  },
  ui: {
    temperatureType: 'range',
    temperatureOptions: {
      min: 0.0,
      max: 1.0,
      step: 0.1,
      default: 0.7
    },
    displayOrder: 20,
    color: '#4285F4',
    category: 'local'
  },
  status: {
    isAvailable: false,
    requiresLoading: true,
    loadingSupported: true,
    healthStatus: 'unknown'
  },
  features: {
    streamingSupported: true,
    summaryGeneration: false,
    titleGeneration: false
  },
  schemaVersion: 1
}
```

---

## 7. Design Principles

### 7.1 Data Validation

**Validation Strategy:**
- Client-side validation for immediate user feedback
- Server-side validation for security and data integrity
- Schema-based validation using TypeScript interfaces
- Custom validation rules for business logic

**Key Validation Areas:**
- User input sanitization (prevent XSS, injection attacks)
- API key format validation before encryption
- Message content length and format validation
- Model configuration parameter validation

### 7.2 Security & Encryption

**Security Principles:**
- All API keys encrypted using AES-256-GCM
- User passwords hashed with bcrypt (cost factor 12+)
- Content sanitization for all user inputs
- Role-based access control for all operations
- Audit logging for sensitive operations

**Data Protection:**
- API keys encrypted with user-specific salts
- Automatic key rotation capabilities
- Secure key storage with hash verification
- No plaintext sensitive data in database

### 7.3 Performance Optimization

**Optimization Strategies:**
- Cursor-based pagination for all list operations
- Multi-layer caching (memory + persistent)
- Denormalized data for frequently accessed information
- Optimized Firebase queries with proper indexing
- Batch operations for bulk data processing

**Caching Strategy:**
- User profiles cached for 5 minutes
- Model configurations cached for 30 minutes
- Thread lists cached with invalidation on updates
- Message tokens cached during streaming

### 7.4 Backup & Recovery

**Backup Strategy:**
- Daily automated backups to cloud storage
- Incremental backups for large datasets
- Encrypted backup files with compression
- Multiple backup destinations for redundancy
- Point-in-time recovery using transaction logs

**Recovery Procedures:**
- Automated backup verification
- Staged recovery testing
- Rollback procedures for failed updates
- Data consistency checks post-recovery

---

This design document provides the foundation for implementing a robust, scalable, and secure database layer for the LLM Chat application using Firebase Realtime Database. The modular design supports real-time operations while maintaining data integrity and security standards.