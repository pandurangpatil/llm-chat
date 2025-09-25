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
└── models_config/            # Model catalog and configuration
```

### Design Principles

- Optimize for read operations and real-time updates
- Atomic updates within document boundaries
- Cursor-based pagination for all lists
- Firebase real-time listeners support
- Encrypted API keys with access control

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
  createAt: number;              // Creation timestamp (epoch)
  updateAt: number;              // Last update timestamp (epoch)
  lastLoginAt?: number;          // Last successful login

  // Settings
  settings: {
    defaultModel: string;        // Default model ID
    defaultTemperature: number;  // Default temperature (0-1)
    systemPrompt: string;        // User's default system prompt
    summarizationInterval: number; // Number of messages before auto-summarization (default: 10)
  };

  // Encrypted API Keys
  apiKeys: {
    claude?: {
      encryptedKey: string;      // AES-256-GCM encrypted
      keyHash: string;           // SHA-256 hash for verification
    };
    openai?: {
      encryptedKey: string;
      keyHash: string;
    };
    google?: {
      encryptedKey: string;
      keyHash: string;
    };
  };
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
  createAt: number;              // Creation timestamp (epoch)
  updateAt: number;              // Last activity timestamp (epoch)

  // Model-specific tracking
  models: {
    [modelId: string]: {
      messageCount: number;      // Messages for this model
      lastMessageAt: number;     // Last message timestamp
      summary?: string;          // AI-generated summary (300-700 tokens)
      summaryUpdatedAt?: number; // Summary generation timestamp
      summaryTokens?: number;    // Token count of summary
      summaryJobStatus?: 'pending' | 'generating' | 'complete' | 'failed'; // Summarization job status
      lastTemperature?: number;  // Last used temperature
      totalTokensUsed: number;   // Total tokens consumed
      status: 'active' | 'archived' | 'error';
    };
  };
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

  // Summarization tracking
  triggeredSummarization?: boolean;  // Whether this message triggered auto-summarization
  summarizationJobId?: string;       // Reference to summarization job (for tracking)

  // Timestamps
  createAt: number;              // Message creation time (epoch)
  updateAt: number;              // Last update time (epoch)
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

  // Timestamps
  createAt: number;              // Creation timestamp (epoch)
  updateAt: number;              // Last update timestamp (epoch)
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

Models_Config (1) ── (N) Messages (via modelId)
Threads (1) ─────── (N) Messages (via threadId)
```

### 3.2 Referential Integrity Rules

#### User Deletion
- **CASCADE**: Delete all user's threads and messages
- **ENCRYPT**: Overwrite API keys with random data before deletion

#### Thread Deletion
- **CASCADE**: Delete all messages within the thread

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
      ".indexOn": ["username", "lastLoginAt"]
    },

    // Thread queries by user with pagination
    "threads": {
      ".indexOn": ["userId", "updateAt", "createAt"]
    },

    // Message queries by thread and model
    "messages": {
      ".indexOn": ["threadId", "modelId", "createAt", "status", "userId"]
    },

    // Model config queries
    "models_config": {
      ".indexOn": ["provider", "ui.displayOrder"]
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
  .orderByChild('updateAt')
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
  .orderByChild('createAt')
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
- Initialize empty collections: `users`, `threads`, `messages`
- Populate models_config with supported models

#### Step 3: Model Configuration
Initial model configurations are populated directly into the `models_config` collection during initialization.

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
  createAt: 1640995200000,      // Example timestamp
  updateAt: 1640995200000       // Example timestamp
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
  createAt: 1640995200000,      // Example timestamp
  updateAt: 1640995200000       // Example timestamp
}
```

---

## 7. Implementation

### Data Validation
- Client and server-side validation
- TypeScript schema validation
- Input sanitization (XSS prevention)
- API key format validation

### Security & Encryption
- API keys encrypted with AES-256-GCM
- bcrypt password hashing (cost 12+)
- Content sanitization for all inputs
- Role-based access control


---

This design document provides the foundation for implementing a robust, scalable, and secure database layer for the LLM Chat application using Firebase Realtime Database. The modular design supports real-time operations while maintaining data integrity and security standards.