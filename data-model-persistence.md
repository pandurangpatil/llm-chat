# Data Model & Persistence Architecture

This document provides comprehensive specifications for the data model, persistence layer, and migration system for the LLM Chat application.

## Table of Contents

1. [Database Architecture Overview](#1-database-architecture-overview)
2. [Collection/Table Schemas](#2-collectiontable-schemas)
3. [Data Relationships & Integrity](#3-data-relationships--integrity)
4. [Indexing Strategy](#4-indexing-strategy)
5. [Migration System Architecture](#5-migration-system-architecture)
6. [Data Validation & Constraints](#6-data-validation--constraints)
7. [Performance Optimizations](#7-performance-optimizations)
8. [Security & Encryption](#8-security--encryption)
9. [Backup & Recovery](#9-backup--recovery)
10. [Database Provider Abstraction](#10-database-provider-abstraction)

---

## 1. Database Architecture Overview

### High-Level Structure

The application uses a **document-oriented database design** optimized for Firebase Realtime Database (production) and Firebase Emulator (development). The schema is designed to be provider-agnostic with an abstraction layer supporting future migration to MongoDB or other document databases.

### Top-Level Collections/Nodes

```
Database Root
├── users/                    # User profiles and encrypted API keys
├── threads/                  # Conversation threads with metadata
├── messages/                 # Individual messages (separate collection)
├── models_config/            # Model catalog and configuration
├── operations/               # Async operation tracking
├── system/                   # Application metadata and version info
└── migrations/               # Migration version tracking
```

### Design Principles

- **Denormalization for Performance**: Optimize for read operations and real-time updates
- **Atomic Operations**: Design updates to be atomic within document boundaries
- **Scalable Pagination**: Use cursor-based pagination for all list operations
- **Real-time Capable**: Structure supports Firebase real-time listeners
- **Migration-Friendly**: Schema versioning and evolution support

---

## 2. Collection/Table Schemas

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
  schemaVersion: number;         // For migration tracking
}
```

**Validation Rules:**
- `username`: Must be valid email format, unique across collection
- `passwordHash`: Minimum bcrypt cost factor of 12
- `displayName`: 1-100 characters, no special characters
- `settings.defaultTemperature`: Range 0.0-1.0
- `settings.systemPrompt`: Maximum 2000 characters
- `apiKeys.*.encryptedKey`: Base64 encoded encrypted data
- `schemaVersion`: Current version 1

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

**Validation Rules:**
- `title`: 1-200 characters
- `models[].summary`: Maximum 1000 characters
- `models[].lastTemperature`: Range 0.0-1.0
- `settings.tags`: Maximum 10 tags, each 1-30 characters
- `activeModels`: Must exist in models_config

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

**Validation Rules:**
- `role`: Must be 'user', 'assistant', or 'system'
- `content`: Required for user/system messages, max 100,000 characters
- `tokens`: Array of strings, max 10,000 elements for assistant messages
- `temperature`: Range 0.0-1.0
- `systemPrompt`: Max 2000 characters
- `status`: Must be valid enum value
- `feedback.rating`: Integer 1-5
- `feedback.comments`: Max 1000 characters

### 2.4 Models Config Collection

**Purpose**: Centralized model catalog with capabilities and configuration.

```typescript
interface ModelConfigDocument {
  // Model identity
  id: string;                    // Primary key: "claude-opus", "gemma-2b-local"
  name: string;                  // Display name: "Claude 3 Opus"
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

**Validation Rules:**
- `name`: 1-100 characters
- `provider`: Must be valid enum value
- `specs.contextLength`: Positive integer
- `specs.*TokenCost`: Non-negative number
- `ui.temperatureOptions.min/max`: Range 0.0-1.0
- `ui.displayOrder`: Non-negative integer
- `status.errorRate`: Range 0.0-100.0

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

**Validation Rules:**
- `type`: Must be valid enum value
- `status`: Must be valid enum value
- `progress`: Range 0-100
- `timeoutAt`: Must be future timestamp
- `error.retryCount`: Non-negative integer

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

### 2.7 Migrations Collection

**Purpose**: Track database migrations and schema versioning.

```typescript
interface MigrationDocument {
  // Migration identity
  id: string;                    // Primary key: "001_initial_schema"
  version: number;               // Migration version number
  name: string;                  // Descriptive migration name

  // Migration details
  description: string;           // What this migration does
  scriptPath?: string;           // Path to migration script
  checksum: string;              // MD5 hash of migration content

  // Execution tracking
  status: 'pending' | 'running' | 'completed' | 'failed' | 'rolled_back';
  executedAt?: number;           // Execution timestamp
  executionTime?: number;        // Execution duration in ms
  appliedBy: string;             // System or user who applied

  // Rollback information
  rollbackScript?: string;       // Rollback script content
  canRollback: boolean;          // Whether rollback is safe
  rolledBackAt?: number;         // Rollback timestamp
  rolledBackBy?: string;         // Who performed rollback

  // Error handling
  error?: {
    message: string;
    stack?: string;
    details?: any;
  };

  // Dependencies
  dependencies?: string[];       // Required prior migrations

  // Metadata
  createdAt: number;

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

#### Message Token Streaming
```typescript
// Atomic token append operation
const atomicTokenAppend = {
  operation: 'array-append',
  path: 'messages/{messageId}/tokens',
  value: newToken,
  conditions: {
    'status': 'generating',
    'tokens.length': { $lt: 10000 } // Prevent runaway generation
  }
};
```

#### Thread Statistics Updates
```typescript
// Atomic counter updates
const updateThreadStats = {
  operation: 'increment',
  path: 'threads/{threadId}/models/{modelId}',
  updates: {
    'messageCount': 1,
    'totalTokensUsed': tokenCount,
    'lastMessageAt': timestamp
  }
};
```

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
    },

    // Migration tracking
    "migrations": {
      ".indexOn": ["version", "status", "executedAt"]
    }
  }
}
```

### 4.2 MongoDB Indexes (Alternative Provider)

```javascript
// Users collection indexes
db.users.createIndex({ "username": 1 }, { unique: true });
db.users.createIndex({ "status": 1, "lastLoginAt": -1 });
db.users.createIndex({ "apiKeys.claude.lastUsed": -1 });

// Threads collection indexes
db.threads.createIndex({ "userId": 1, "updatedAt": -1 });
db.threads.createIndex({ "userId": 1, "settings.isArchived": 1, "updatedAt": -1 });
db.threads.createIndex({ "activeModels": 1 });

// Messages collection indexes
db.messages.createIndex({ "threadId": 1, "createdAt": -1 });
db.messages.createIndex({ "threadId": 1, "modelId": 1, "createdAt": -1 });
db.messages.createIndex({ "userId": 1, "status": 1 });
db.messages.createIndex({ "status": 1, "createdAt": 1 }); // For cleanup jobs

// Text search indexes
db.threads.createIndex({
  "title": "text",
  "models.$**.summary": "text"
}, {
  weights: { "title": 2, "models.$**.summary": 1 }
});

// Models config indexes
db.models_config.createIndex({ "provider": 1, "status.isAvailable": 1 });
db.models_config.createIndex({ "ui.displayOrder": 1 });

// Operations indexes
db.operations.createIndex({ "userId": 1, "status": 1, "createdAt": -1 });
db.operations.createIndex({ "status": 1, "timeoutAt": 1 }); // For timeout cleanup
db.operations.createIndex({ "type": 1, "status": 1 });

// Migrations indexes
db.migrations.createIndex({ "version": 1 }, { unique: true });
db.migrations.createIndex({ "status": 1, "executedAt": -1 });
```

### 4.3 Query Optimization Patterns

#### Common Query Examples

**Thread List with Pagination:**
```typescript
// Firebase query
const threadsQuery = database
  .ref('threads')
  .orderByChild('userId')
  .equalTo(userId)
  .orderByChild('updatedAt')
  .limitToLast(20);

// MongoDB query
db.threads.find({ userId })
  .sort({ updatedAt: -1 })
  .limit(20);
```

**Message History for Thread/Model:**
```typescript
// Firebase query
const messagesQuery = database
  .ref('messages')
  .orderByChild('threadId')
  .equalTo(threadId)
  .orderByChild('modelId')
  .equalTo(modelId)
  .orderByChild('createdAt')
  .limitToLast(50);

// MongoDB query with compound index
db.messages.find({ threadId, modelId })
  .sort({ createdAt: -1 })
  .limit(50);
```

---

## 5. Migration System Architecture

### 5.1 Migration Framework Design

The migration system supports schema evolution and data transformation without external dependencies, providing a robust foundation for database changes throughout the application lifecycle.

#### Core Migration Architecture

```typescript
interface MigrationRunner {
  // Migration discovery and validation
  discoverMigrations(): Promise<Migration[]>;
  validateMigration(migration: Migration): Promise<boolean>;

  // Execution management
  runPendingMigrations(): Promise<MigrationResult[]>;
  runSpecificMigration(version: number): Promise<MigrationResult>;
  rollbackMigration(version: number): Promise<MigrationResult>;

  // Status and tracking
  getCurrentVersion(): Promise<number>;
  getMigrationStatus(): Promise<MigrationStatus[]>;

  // Safety and locking
  acquireLock(): Promise<boolean>;
  releaseLock(): Promise<void>;
}
```

### 5.2 Migration File Structure

#### Directory Organization
```
migrations/
├── 001_initial_schema.js        # Initial database setup
├── 002_add_user_settings.js     # Add user settings field
├── 003_models_config_table.js   # Create models config
├── 004_message_streaming.js     # Add streaming support
├── 005_operations_tracking.js   # Add operations table
├── lib/
│   ├── migration-runner.js      # Core migration engine
│   ├── validators.js            # Schema validation helpers
│   └── helpers.js               # Common migration utilities
└── seeds/
    ├── development.js           # Development seed data
    ├── test.js                  # Test environment seeds
    └── production.js            # Production seed data
```

#### Migration File Format

```javascript
// migrations/002_add_user_settings.js
module.exports = {
  version: 2,
  name: 'add_user_settings',
  description: 'Add settings field to user documents',

  // Forward migration
  async up(db, helpers) {
    const users = await db.collection('users').getAll();

    for (const user of users) {
      await db.collection('users').doc(user.id).update({
        settings: {
          defaultModel: 'claude-opus',
          defaultTemperature: 0.7,
          systemPrompt: 'You are a helpful AI assistant.',
          streamingEnabled: true
        },
        schemaVersion: 2
      });
    }

    // Update system version
    await db.collection('system').doc('version').update({
      schema: 2,
      lastMigration: Date.now()
    });
  },

  // Rollback migration
  async down(db, helpers) {
    const users = await db.collection('users').getAll();

    for (const user of users) {
      await db.collection('users').doc(user.id).update({
        settings: null,
        schemaVersion: 1
      });
    }

    await db.collection('system').doc('version').update({
      schema: 1,
      lastMigration: Date.now()
    });
  },

  // Validation checks
  async validate(db) {
    const users = await db.collection('users').getAll();

    for (const user of users) {
      if (!user.settings || typeof user.settings !== 'object') {
        throw new Error(`User ${user.id} missing settings field`);
      }

      if (user.schemaVersion !== 2) {
        throw new Error(`User ${user.id} has incorrect schema version`);
      }
    }

    return true;
  }
};
```

### 5.3 Migration Runner Implementation

#### Core Migration Engine

```javascript
// lib/migration-runner.js
class MigrationRunner {
  constructor(database, logger) {
    this.db = database;
    this.logger = logger;
    this.lockTimeout = 300000; // 5 minutes
  }

  async runPendingMigrations() {
    try {
      // Acquire distributed lock
      const lockAcquired = await this.acquireLock();
      if (!lockAcquired) {
        throw new Error('Could not acquire migration lock');
      }

      const currentVersion = await this.getCurrentVersion();
      const allMigrations = await this.discoverMigrations();
      const pendingMigrations = allMigrations.filter(m => m.version > currentVersion);

      const results = [];

      for (const migration of pendingMigrations) {
        this.logger.info(`Running migration ${migration.version}: ${migration.name}`);

        try {
          // Record migration start
          await this.recordMigrationStart(migration);

          // Execute migration
          const startTime = Date.now();
          await migration.up(this.db, this.getMigrationHelpers());
          const duration = Date.now() - startTime;

          // Validate migration result
          if (migration.validate) {
            await migration.validate(this.db);
          }

          // Record success
          await this.recordMigrationComplete(migration, duration);

          results.push({
            version: migration.version,
            status: 'completed',
            duration
          });

        } catch (error) {
          this.logger.error(`Migration ${migration.version} failed:`, error);

          // Record failure
          await this.recordMigrationError(migration, error);

          // Attempt rollback if available
          if (migration.down) {
            try {
              await migration.down(this.db, this.getMigrationHelpers());
              this.logger.info(`Rolled back migration ${migration.version}`);
            } catch (rollbackError) {
              this.logger.error(`Rollback failed for migration ${migration.version}:`, rollbackError);
            }
          }

          results.push({
            version: migration.version,
            status: 'failed',
            error: error.message
          });

          break; // Stop on first failure
        }
      }

      return results;

    } finally {
      await this.releaseLock();
    }
  }

  async getCurrentVersion() {
    try {
      const versionDoc = await this.db.collection('system').doc('version').get();
      return versionDoc?.schema || 0;
    } catch (error) {
      // If system collection doesn't exist, assume version 0
      return 0;
    }
  }

  async acquireLock() {
    const lockDoc = {
      locked: true,
      lockedAt: Date.now(),
      lockedBy: process.env.HOSTNAME || 'unknown',
      expiresAt: Date.now() + this.lockTimeout
    };

    try {
      // Attempt atomic lock creation
      await this.db.collection('system').doc('migration_lock').create(lockDoc);
      return true;
    } catch (error) {
      // Check if lock is expired
      const existingLock = await this.db.collection('system').doc('migration_lock').get();

      if (existingLock && existingLock.expiresAt < Date.now()) {
        // Lock expired, force acquire
        await this.db.collection('system').doc('migration_lock').update(lockDoc);
        return true;
      }

      return false;
    }
  }

  async releaseLock() {
    try {
      await this.db.collection('system').doc('migration_lock').delete();
    } catch (error) {
      this.logger.warn('Failed to release migration lock:', error);
    }
  }

  getMigrationHelpers() {
    return {
      // Collection helpers
      async ensureCollection(name) {
        // Implementation depends on database provider
      },

      // Index helpers
      async createIndex(collection, fields, options = {}) {
        // Implementation depends on database provider
      },

      // Data transformation helpers
      async batchUpdate(collection, updates, batchSize = 100) {
        const docs = await this.db.collection(collection).getAll();

        for (let i = 0; i < docs.length; i += batchSize) {
          const batch = docs.slice(i, i + batchSize);
          const promises = batch.map(doc =>
            this.db.collection(collection).doc(doc.id).update(updates[doc.id])
          );
          await Promise.all(promises);
        }
      },

      // Validation helpers
      async validateSchema(collection, validator) {
        const docs = await this.db.collection(collection).getAll();
        for (const doc of docs) {
          if (!validator(doc)) {
            throw new Error(`Document ${doc.id} failed validation`);
          }
        }
      }
    };
  }
}
```

### 5.4 Startup Integration

#### Application Bootstrap

```javascript
// app.js - Application startup
const MigrationRunner = require('./migrations/lib/migration-runner');

async function startApplication() {
  try {
    // Initialize database connection
    const database = await initializeDatabase();

    // Run migrations if enabled
    if (process.env.RUN_MIGRATIONS === 'true') {
      const migrationRunner = new MigrationRunner(database, logger);

      logger.info('Running database migrations...');
      const results = await migrationRunner.runPendingMigrations();

      for (const result of results) {
        if (result.status === 'completed') {
          logger.info(`Migration ${result.version} completed in ${result.duration}ms`);
        } else {
          logger.error(`Migration ${result.version} failed: ${result.error}`);
          process.exit(1);
        }
      }

      logger.info('All migrations completed successfully');
    }

    // Start application services
    await startExpressServer();

  } catch (error) {
    logger.error('Application startup failed:', error);
    process.exit(1);
  }
}
```

### 5.5 Migration Commands

#### CLI Interface

```javascript
// scripts/migrate.js
const { program } = require('commander');
const MigrationRunner = require('../migrations/lib/migration-runner');

program
  .command('status')
  .description('Show migration status')
  .action(async () => {
    const runner = new MigrationRunner(await initDb(), console);
    const status = await runner.getMigrationStatus();
    console.table(status);
  });

program
  .command('up [version]')
  .description('Run migrations up to specified version')
  .action(async (version) => {
    const runner = new MigrationRunner(await initDb(), console);

    if (version) {
      await runner.runSpecificMigration(parseInt(version));
    } else {
      await runner.runPendingMigrations();
    }
  });

program
  .command('down <version>')
  .description('Rollback to specified version')
  .action(async (version) => {
    const runner = new MigrationRunner(await initDb(), console);
    await runner.rollbackMigration(parseInt(version));
  });

program
  .command('create <name>')
  .description('Create new migration file')
  .action(async (name) => {
    const timestamp = Date.now();
    const version = await getNextVersion();
    const filename = `${version.toString().padStart(3, '0')}_${name}.js`;

    const template = `module.exports = {
  version: ${version},
  name: '${name}',
  description: 'Add description here',

  async up(db, helpers) {
    // Forward migration logic
  },

  async down(db, helpers) {
    // Rollback logic
  },

  async validate(db) {
    // Validation checks
    return true;
  }
};`;

    await fs.writeFile(`migrations/${filename}`, template);
    console.log(`Created migration: ${filename}`);
  });

program.parse();
```

---

## 6. Data Validation & Constraints

### 6.1 Schema Validation Framework

#### Validation Engine

```typescript
interface ValidationRule {
  field: string;
  type: 'required' | 'type' | 'range' | 'length' | 'pattern' | 'custom';
  constraint: any;
  message: string;
}

interface ValidationSchema {
  collection: string;
  version: number;
  rules: ValidationRule[];
}

class SchemaValidator {
  constructor(private schemas: Map<string, ValidationSchema>) {}

  async validate(collection: string, document: any): Promise<ValidationResult> {
    const schema = this.schemas.get(collection);
    if (!schema) {
      throw new Error(`No schema defined for collection: ${collection}`);
    }

    const errors: ValidationError[] = [];

    for (const rule of schema.rules) {
      const result = await this.validateRule(document, rule);
      if (!result.valid) {
        errors.push({
          field: rule.field,
          message: rule.message,
          value: result.value
        });
      }
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }

  private async validateRule(document: any, rule: ValidationRule): Promise<RuleResult> {
    const value = this.getFieldValue(document, rule.field);

    switch (rule.type) {
      case 'required':
        return {
          valid: value !== undefined && value !== null && value !== '',
          value
        };

      case 'type':
        return {
          valid: typeof value === rule.constraint,
          value
        };

      case 'range':
        const [min, max] = rule.constraint;
        return {
          valid: typeof value === 'number' && value >= min && value <= max,
          value
        };

      case 'length':
        const [minLen, maxLen] = rule.constraint;
        const length = value?.length || 0;
        return {
          valid: length >= minLen && length <= maxLen,
          value
        };

      case 'pattern':
        const regex = new RegExp(rule.constraint);
        return {
          valid: typeof value === 'string' && regex.test(value),
          value
        };

      case 'custom':
        return await rule.constraint(value, document);

      default:
        throw new Error(`Unknown validation rule type: ${rule.type}`);
    }
  }
}
```

### 6.2 Collection-Specific Validation Schemas

#### User Document Validation

```typescript
const userValidationSchema: ValidationSchema = {
  collection: 'users',
  version: 1,
  rules: [
    {
      field: 'username',
      type: 'required',
      constraint: true,
      message: 'Username is required'
    },
    {
      field: 'username',
      type: 'pattern',
      constraint: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$',
      message: 'Username must be a valid email address'
    },
    {
      field: 'displayName',
      type: 'length',
      constraint: [1, 100],
      message: 'Display name must be 1-100 characters'
    },
    {
      field: 'passwordHash',
      type: 'required',
      constraint: true,
      message: 'Password hash is required'
    },
    {
      field: 'settings.defaultTemperature',
      type: 'range',
      constraint: [0.0, 1.0],
      message: 'Default temperature must be between 0.0 and 1.0'
    },
    {
      field: 'settings.systemPrompt',
      type: 'length',
      constraint: [0, 2000],
      message: 'System prompt must not exceed 2000 characters'
    },
    {
      field: 'apiKeys',
      type: 'custom',
      constraint: async (value, document) => {
        if (!value || typeof value !== 'object') {
          return { valid: true, value }; // Optional field
        }

        // Validate each API key structure
        for (const [provider, keyData] of Object.entries(value)) {
          if (!['claude', 'openai', 'google'].includes(provider)) {
            return {
              valid: false,
              value,
              message: `Invalid API key provider: ${provider}`
            };
          }

          if (!keyData.encryptedKey || !keyData.keyHash) {
            return {
              valid: false,
              value,
              message: `Missing encryption data for ${provider} API key`
            };
          }
        }

        return { valid: true, value };
      },
      message: 'Invalid API key structure'
    }
  ]
};
```

#### Message Document Validation

```typescript
const messageValidationSchema: ValidationSchema = {
  collection: 'messages',
  version: 1,
  rules: [
    {
      field: 'threadId',
      type: 'required',
      constraint: true,
      message: 'Thread ID is required'
    },
    {
      field: 'modelId',
      type: 'required',
      constraint: true,
      message: 'Model ID is required'
    },
    {
      field: 'role',
      type: 'pattern',
      constraint: '^(user|assistant|system)$',
      message: 'Role must be user, assistant, or system'
    },
    {
      field: 'status',
      type: 'pattern',
      constraint: '^(pending|generating|complete|failed|cancelled)$',
      message: 'Invalid message status'
    },
    {
      field: 'content',
      type: 'custom',
      constraint: async (value, document) => {
        // Content required for user/system messages
        if (document.role !== 'assistant' && (!value || value.trim() === '')) {
          return {
            valid: false,
            value,
            message: 'Content is required for user and system messages'
          };
        }

        // Content length validation
        if (value && value.length > 100000) {
          return {
            valid: false,
            value,
            message: 'Content exceeds maximum length of 100,000 characters'
          };
        }

        return { valid: true, value };
      },
      message: 'Invalid message content'
    },
    {
      field: 'tokens',
      type: 'custom',
      constraint: async (value, document) => {
        // Tokens only for assistant messages
        if (document.role === 'assistant') {
          if (!Array.isArray(value)) {
            return { valid: true, value }; // Allow null/undefined for incomplete messages
          }

          if (value.length > 10000) {
            return {
              valid: false,
              value,
              message: 'Token array exceeds maximum length of 10,000'
            };
          }

          // Validate token types
          for (const token of value) {
            if (typeof token !== 'string') {
              return {
                valid: false,
                value,
                message: 'All tokens must be strings'
              };
            }
          }
        } else if (value !== undefined && value !== null) {
          return {
            valid: false,
            value,
            message: 'Tokens array only allowed for assistant messages'
          };
        }

        return { valid: true, value };
      },
      message: 'Invalid tokens array'
    },
    {
      field: 'temperature',
      type: 'custom',
      constraint: async (value, document) => {
        if (value === undefined || value === null) {
          return { valid: true, value }; // Optional field
        }

        if (typeof value !== 'number' || value < 0.0 || value > 1.0) {
          return {
            valid: false,
            value,
            message: 'Temperature must be a number between 0.0 and 1.0'
          };
        }

        return { valid: true, value };
      },
      message: 'Invalid temperature value'
    }
  ]
};
```

### 6.3 Runtime Validation Integration

#### Database Layer Integration

```typescript
class ValidatedDatabase {
  constructor(
    private database: Database,
    private validator: SchemaValidator
  ) {}

  async create(collection: string, document: any): Promise<string> {
    // Validate before creating
    const validation = await this.validator.validate(collection, document);
    if (!validation.valid) {
      throw new ValidationError(
        `Validation failed for ${collection}`,
        validation.errors
      );
    }

    return this.database.create(collection, document);
  }

  async update(collection: string, id: string, updates: any): Promise<void> {
    // Get existing document for full validation
    const existing = await this.database.get(collection, id);
    const merged = { ...existing, ...updates };

    const validation = await this.validator.validate(collection, merged);
    if (!validation.valid) {
      throw new ValidationError(
        `Validation failed for ${collection}/${id}`,
        validation.errors
      );
    }

    return this.database.update(collection, id, updates);
  }
}
```

#### API Layer Validation

```typescript
// Express middleware for request validation
function validateRequest(schema: ValidationSchema) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      const validator = new SchemaValidator(new Map([[schema.collection, schema]]));
      const validation = await validator.validate(schema.collection, req.body);

      if (!validation.valid) {
        return res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            message: 'Request validation failed',
            details: validation.errors
          }
        });
      }

      next();
    } catch (error) {
      next(error);
    }
  };
}

// Usage in routes
app.post('/api/users',
  validateRequest(userValidationSchema),
  async (req, res) => {
    // Handler logic
  }
);
```

---

## 7. Performance Optimizations

### 7.1 Query Optimization Strategies

#### Cursor-Based Pagination Implementation

```typescript
interface PaginationCursor {
  field: string;
  value: any;
  direction: 'asc' | 'desc';
}

class PaginatedQuery {
  constructor(
    private collection: string,
    private orderField: string = 'createdAt',
    private direction: 'asc' | 'desc' = 'desc'
  ) {}

  async execute(
    limit: number = 20,
    cursor?: string,
    filters: Record<string, any> = {}
  ): Promise<PaginationResult> {
    let query = this.database.collection(this.collection);

    // Apply filters
    for (const [field, value] of Object.entries(filters)) {
      query = query.where(field, '==', value);
    }

    // Apply cursor for pagination
    if (cursor) {
      const decodedCursor = this.decodeCursor(cursor);
      const operator = this.direction === 'desc' ? '<' : '>';
      query = query.where(this.orderField, operator, decodedCursor.value);
    }

    // Apply ordering and limit
    query = query.orderBy(this.orderField, this.direction).limit(limit + 1);

    const results = await query.get();
    const hasMore = results.length > limit;
    const items = hasMore ? results.slice(0, -1) : results;

    const nextCursor = hasMore
      ? this.encodeCursor({
          field: this.orderField,
          value: items[items.length - 1][this.orderField],
          direction: this.direction
        })
      : null;

    return {
      items,
      nextCursor,
      hasMore,
      totalCount: await this.getTotalCount(filters)
    };
  }

  private encodeCursor(cursor: PaginationCursor): string {
    return Buffer.from(JSON.stringify(cursor)).toString('base64');
  }

  private decodeCursor(encoded: string): PaginationCursor {
    return JSON.parse(Buffer.from(encoded, 'base64').toString());
  }
}
```

#### Optimized Thread Queries

```typescript
class ThreadQueryOptimizer {
  async getThreadsWithPagination(
    userId: string,
    options: ThreadQueryOptions = {}
  ): Promise<PaginationResult<ThreadSummary>> {
    const {
      limit = 20,
      cursor,
      search,
      includeArchived = false,
      orderBy = 'updatedAt'
    } = options;

    // Use compound index: userId + orderBy + archived status
    let query = this.database
      .collection('threads')
      .where('userId', '==', userId);

    if (!includeArchived) {
      query = query.where('settings.isArchived', '==', false);
    }

    if (search) {
      // Use text search index when available
      query = query.where('title', '>=', search)
                  .where('title', '<=', search + '\uf8ff');
    }

    const paginated = new PaginatedQuery('threads', orderBy);
    const result = await paginated.execute(limit, cursor, {
      userId,
      'settings.isArchived': includeArchived ? undefined : false
    });

    // Transform to summary format (reduce data transfer)
    const threadSummaries = result.items.map(thread => ({
      id: thread.id,
      title: thread.title,
      createdAt: thread.createdAt,
      updatedAt: thread.updatedAt,
      messageCount: thread.totalMessages,
      activeModels: thread.activeModels,
      lastMessage: this.getLastMessageSummary(thread)
    }));

    return {
      ...result,
      items: threadSummaries
    };
  }

  private getLastMessageSummary(thread: ThreadDocument): MessageSummary | null {
    let lastMessage = null;
    let lastTimestamp = 0;

    // Find most recent message across all models
    for (const [modelId, modelData] of Object.entries(thread.models)) {
      if (modelData.lastMessageAt > lastTimestamp) {
        lastTimestamp = modelData.lastMessageAt;
        lastMessage = {
          modelId,
          timestamp: modelData.lastMessageAt,
          // Note: We don't store message content in thread document
          // This would require a separate query if needed
        };
      }
    }

    return lastMessage;
  }
}
```

### 7.2 Caching Strategies

#### Multi-Layer Caching Architecture

```typescript
interface CacheLayer {
  get(key: string): Promise<any>;
  set(key: string, value: any, ttl?: number): Promise<void>;
  delete(key: string): Promise<void>;
  clear(pattern?: string): Promise<void>;
}

class MemoryCache implements CacheLayer {
  private cache = new Map<string, CacheEntry>();
  private maxSize: number;

  constructor(maxSize: number = 1000) {
    this.maxSize = maxSize;
  }

  async get(key: string): Promise<any> {
    const entry = this.cache.get(key);
    if (!entry) return null;

    if (entry.expiresAt && entry.expiresAt < Date.now()) {
      this.cache.delete(key);
      return null;
    }

    // Update access time for LRU
    entry.accessedAt = Date.now();
    return entry.value;
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    // Implement LRU eviction if cache is full
    if (this.cache.size >= this.maxSize) {
      this.evictLRU();
    }

    const expiresAt = ttl ? Date.now() + ttl * 1000 : null;
    this.cache.set(key, {
      value,
      createdAt: Date.now(),
      accessedAt: Date.now(),
      expiresAt
    });
  }

  private evictLRU(): void {
    let oldestKey = null;
    let oldestTime = Date.now();

    for (const [key, entry] of this.cache.entries()) {
      if (entry.accessedAt < oldestTime) {
        oldestTime = entry.accessedAt;
        oldestKey = key;
      }
    }

    if (oldestKey) {
      this.cache.delete(oldestKey);
    }
  }
}

class CachedDatabase {
  constructor(
    private database: Database,
    private cache: CacheLayer,
    private defaultTTL: number = 300 // 5 minutes
  ) {}

  async get(collection: string, id: string): Promise<any> {
    const cacheKey = `${collection}:${id}`;

    // Try cache first
    let result = await this.cache.get(cacheKey);
    if (result) {
      return result;
    }

    // Fallback to database
    result = await this.database.get(collection, id);
    if (result) {
      await this.cache.set(cacheKey, result, this.defaultTTL);
    }

    return result;
  }

  async update(collection: string, id: string, updates: any): Promise<void> {
    // Update database
    await this.database.update(collection, id, updates);

    // Invalidate cache
    const cacheKey = `${collection}:${id}`;
    await this.cache.delete(cacheKey);

    // Optionally warm cache with updated data
    const updated = await this.database.get(collection, id);
    await this.cache.set(cacheKey, updated, this.defaultTTL);
  }
}
```

#### Model Status Caching

```typescript
class ModelStatusCache {
  private statusCache = new Map<string, ModelStatusEntry>();
  private readonly STATUS_TTL = 30000; // 30 seconds

  async getModelStatus(modelId: string): Promise<ModelStatus> {
    const cached = this.statusCache.get(modelId);

    if (cached && (Date.now() - cached.timestamp) < this.STATUS_TTL) {
      return cached.status;
    }

    // Fetch fresh status
    const status = await this.fetchModelStatus(modelId);

    this.statusCache.set(modelId, {
      status,
      timestamp: Date.now()
    });

    return status;
  }

  private async fetchModelStatus(modelId: string): Promise<ModelStatus> {
    // Implementation depends on model provider
    if (modelId.includes('local')) {
      return this.getOllamaModelStatus(modelId);
    } else {
      return this.getCloudModelStatus(modelId);
    }
  }

  invalidateModel(modelId: string): void {
    this.statusCache.delete(modelId);
  }

  // Periodic cleanup of stale entries
  cleanup(): void {
    const now = Date.now();
    for (const [modelId, entry] of this.statusCache.entries()) {
      if ((now - entry.timestamp) > this.STATUS_TTL * 2) {
        this.statusCache.delete(modelId);
      }
    }
  }
}
```

### 7.3 Connection Pooling & Resource Management

#### Database Connection Management

```typescript
class ConnectionPool {
  private connections: Connection[] = [];
  private availableConnections: Connection[] = [];
  private readonly maxConnections: number;
  private readonly minConnections: number;

  constructor(options: PoolOptions) {
    this.maxConnections = options.max || 10;
    this.minConnections = options.min || 2;
  }

  async initialize(): Promise<void> {
    // Create minimum connections
    for (let i = 0; i < this.minConnections; i++) {
      const connection = await this.createConnection();
      this.connections.push(connection);
      this.availableConnections.push(connection);
    }
  }

  async acquire(): Promise<Connection> {
    // Return available connection if exists
    if (this.availableConnections.length > 0) {
      return this.availableConnections.pop()!;
    }

    // Create new connection if under max limit
    if (this.connections.length < this.maxConnections) {
      const connection = await this.createConnection();
      this.connections.push(connection);
      return connection;
    }

    // Wait for connection to become available
    return this.waitForConnection();
  }

  release(connection: Connection): void {
    if (connection.isHealthy()) {
      this.availableConnections.push(connection);
    } else {
      // Remove unhealthy connection
      const index = this.connections.indexOf(connection);
      if (index > -1) {
        this.connections.splice(index, 1);
      }
      connection.close();
    }
  }

  async healthCheck(): Promise<void> {
    const unhealthyConnections = this.connections.filter(conn => !conn.isHealthy());

    for (const conn of unhealthyConnections) {
      const index = this.connections.indexOf(conn);
      if (index > -1) {
        this.connections.splice(index, 1);
      }

      const availableIndex = this.availableConnections.indexOf(conn);
      if (availableIndex > -1) {
        this.availableConnections.splice(availableIndex, 1);
      }

      conn.close();
    }

    // Ensure minimum connections
    while (this.connections.length < this.minConnections) {
      const connection = await this.createConnection();
      this.connections.push(connection);
      this.availableConnections.push(connection);
    }
  }
}
```

### 7.4 Batch Operations

#### Efficient Bulk Operations

```typescript
class BatchProcessor {
  private readonly batchSize: number;
  private readonly maxConcurrency: number;

  constructor(batchSize: number = 100, maxConcurrency: number = 5) {
    this.batchSize = batchSize;
    this.maxConcurrency = maxConcurrency;
  }

  async processBatch<T, R>(
    items: T[],
    processor: (batch: T[]) => Promise<R[]>
  ): Promise<R[]> {
    const results: R[] = [];
    const batches = this.createBatches(items);

    // Process batches with controlled concurrency
    for (let i = 0; i < batches.length; i += this.maxConcurrency) {
      const concurrentBatches = batches.slice(i, i + this.maxConcurrency);
      const batchPromises = concurrentBatches.map(batch => processor(batch));

      const batchResults = await Promise.all(batchPromises);
      results.push(...batchResults.flat());
    }

    return results;
  }

  private createBatches<T>(items: T[]): T[][] {
    const batches: T[][] = [];

    for (let i = 0; i < items.length; i += this.batchSize) {
      batches.push(items.slice(i, i + this.batchSize));
    }

    return batches;
  }
}

// Usage example for bulk message updates
class MessageBulkOperations {
  constructor(
    private database: Database,
    private batchProcessor: BatchProcessor
  ) {}

  async markMessagesComplete(messageIds: string[]): Promise<void> {
    await this.batchProcessor.processBatch(
      messageIds,
      async (batch) => {
        const updates = batch.map(id => ({
          id,
          updates: {
            status: 'complete',
            updatedAt: Date.now()
          }
        }));

        return this.database.batchUpdate('messages', updates);
      }
    );
  }

  async getMessagesByIds(messageIds: string[]): Promise<MessageDocument[]> {
    return this.batchProcessor.processBatch(
      messageIds,
      async (batch) => {
        return this.database.batchGet('messages', batch);
      }
    );
  }
}
```

---

## 8. Security & Encryption

### 8.1 API Key Encryption

#### Encryption Service Implementation

```typescript
import { createCipher, createDecipher, randomBytes, scrypt } from 'crypto';
import { promisify } from 'util';

const scryptAsync = promisify(scrypt);

class EncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly keyDerivationSalt: Buffer;

  constructor(private systemKey: string) {
    // Use a consistent salt for key derivation
    this.keyDerivationSalt = Buffer.from(process.env.ENCRYPTION_SALT || 'default-salt', 'hex');
  }

  async encryptAPIKey(apiKey: string, userSalt?: string): Promise<EncryptedKeyData> {
    try {
      // Derive encryption key from system key + optional user salt
      const derivationInput = userSalt ? `${this.systemKey}:${userSalt}` : this.systemKey;
      const key = await scryptAsync(derivationInput, this.keyDerivationSalt, 32) as Buffer;

      // Generate random IV for each encryption
      const iv = randomBytes(16);

      // Create cipher
      const cipher = createCipher(this.algorithm, key);
      cipher.setAAD(Buffer.from('api-key-data')); // Additional authenticated data

      // Encrypt the API key
      let encrypted = cipher.update(apiKey, 'utf8', 'hex');
      encrypted += cipher.final('hex');

      // Get authentication tag
      const authTag = cipher.getAuthTag();

      // Create key hash for verification (without exposing the key)
      const keyHash = this.createKeyHash(apiKey);

      return {
        encryptedKey: encrypted,
        iv: iv.toString('hex'),
        authTag: authTag.toString('hex'),
        keyHash,
        algorithm: this.algorithm,
        encryptedAt: Date.now()
      };

    } catch (error) {
      throw new Error(`API key encryption failed: ${error.message}`);
    }
  }

  async decryptAPIKey(encryptedData: EncryptedKeyData, userSalt?: string): Promise<string> {
    try {
      // Derive the same encryption key
      const derivationInput = userSalt ? `${this.systemKey}:${userSalt}` : this.systemKey;
      const key = await scryptAsync(derivationInput, this.keyDerivationSalt, 32) as Buffer;

      // Create decipher
      const decipher = createDecipher(this.algorithm, key);
      decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
      decipher.setAAD(Buffer.from('api-key-data'));

      // Decrypt the API key
      let decrypted = decipher.update(encryptedData.encryptedKey, 'hex', 'utf8');
      decrypted += decipher.final('utf8');

      // Verify the decrypted key matches the stored hash
      const computedHash = this.createKeyHash(decrypted);
      if (computedHash !== encryptedData.keyHash) {
        throw new Error('Decrypted API key hash mismatch - possible corruption');
      }

      return decrypted;

    } catch (error) {
      throw new Error(`API key decryption failed: ${error.message}`);
    }
  }

  private createKeyHash(apiKey: string): string {
    const crypto = require('crypto');
    return crypto.createHash('sha256').update(apiKey).digest('hex');
  }

  async rotateEncryption(oldData: EncryptedKeyData, newSystemKey: string): Promise<EncryptedKeyData> {
    // Decrypt with old key
    const plaintext = await this.decryptAPIKey(oldData);

    // Create new encryption service with new key
    const newService = new EncryptionService(newSystemKey);

    // Re-encrypt with new key
    return newService.encryptAPIKey(plaintext);
  }
}

interface EncryptedKeyData {
  encryptedKey: string;
  iv: string;
  authTag: string;
  keyHash: string;
  algorithm: string;
  encryptedAt: number;
}
```

#### Secure API Key Storage

```typescript
class SecureAPIKeyManager {
  constructor(
    private encryption: EncryptionService,
    private database: Database
  ) {}

  async storeUserAPIKey(
    userId: string,
    provider: 'claude' | 'openai' | 'google',
    apiKey: string
  ): Promise<void> {
    // Validate API key format first
    if (!this.validateAPIKeyFormat(provider, apiKey)) {
      throw new Error(`Invalid API key format for provider: ${provider}`);
    }

    // Test API key validity
    const isValid = await this.validateAPIKeyWithProvider(provider, apiKey);
    if (!isValid) {
      throw new Error(`API key validation failed for provider: ${provider}`);
    }

    // Encrypt the API key with user-specific salt
    const userSalt = await this.getUserSalt(userId);
    const encryptedData = await this.encryption.encryptAPIKey(apiKey, userSalt);

    // Store encrypted data
    await this.database.update('users', userId, {
      [`apiKeys.${provider}`]: {
        ...encryptedData,
        isValid: true,
        lastUsed: null,
        createdAt: Date.now()
      },
      updatedAt: Date.now()
    });

    // Log the API key storage (without exposing the key)
    await this.auditLog('api_key_stored', {
      userId,
      provider,
      keyHash: encryptedData.keyHash,
      timestamp: Date.now()
    });
  }

  async getUserAPIKey(userId: string, provider: string): Promise<string | null> {
    const user = await this.database.get('users', userId);
    if (!user?.apiKeys?.[provider]) {
      return null;
    }

    const encryptedData = user.apiKeys[provider];

    try {
      const userSalt = await this.getUserSalt(userId);
      const apiKey = await this.encryption.decryptAPIKey(encryptedData, userSalt);

      // Update last used timestamp
      await this.database.update('users', userId, {
        [`apiKeys.${provider}.lastUsed`]: Date.now()
      });

      return apiKey;

    } catch (error) {
      // Log decryption failure
      await this.auditLog('api_key_decrypt_failed', {
        userId,
        provider,
        error: error.message,
        timestamp: Date.now()
      });

      throw new Error('Failed to decrypt API key');
    }
  }

  private async getUserSalt(userId: string): Promise<string> {
    // Generate consistent salt based on user ID and system secret
    const crypto = require('crypto');
    const saltInput = `${userId}:${process.env.USER_SALT_SECRET}`;
    return crypto.createHash('sha256').update(saltInput).digest('hex').substring(0, 16);
  }

  private validateAPIKeyFormat(provider: string, apiKey: string): boolean {
    const patterns = {
      claude: /^sk-ant-api03-[a-zA-Z0-9_-]{95}$/,
      openai: /^sk-[a-zA-Z0-9]{48}$/,
      google: /^AIza[a-zA-Z0-9_-]{35}$/
    };

    return patterns[provider]?.test(apiKey) || false;
  }

  private async validateAPIKeyWithProvider(provider: string, apiKey: string): Promise<boolean> {
    // Test API key with minimal request to provider
    try {
      switch (provider) {
        case 'claude':
          return await this.testClaudeKey(apiKey);
        case 'openai':
          return await this.testOpenAIKey(apiKey);
        case 'google':
          return await this.testGoogleKey(apiKey);
        default:
          return false;
      }
    } catch (error) {
      return false;
    }
  }

  private async auditLog(action: string, details: any): Promise<void> {
    await this.database.create('audit_logs', {
      action,
      details,
      timestamp: Date.now()
    });
  }
}
```

### 8.2 Data Sanitization & Validation

#### Content Security Measures

```typescript
class ContentSecurityService {
  private readonly sanitizer: any;

  constructor() {
    // Initialize DOMPurify or similar sanitization library
    this.sanitizer = require('isomorphic-dompurify');
  }

  sanitizeUserInput(input: string): string {
    if (!input || typeof input !== 'string') {
      return '';
    }

    // Remove potentially dangerous content
    let sanitized = this.sanitizer.sanitize(input, {
      ALLOWED_TAGS: [
        'p', 'br', 'strong', 'em', 'u', 'ol', 'ul', 'li',
        'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
        'blockquote', 'code', 'pre'
      ],
      ALLOWED_ATTR: ['class'],
      FORBID_SCRIPT: true,
      FORBID_TAGS: ['script', 'object', 'embed', 'link', 'style'],
      STRIP_COMMENTS: true
    });

    // Additional custom sanitization
    sanitized = this.removeExcessiveWhitespace(sanitized);
    sanitized = this.limitLength(sanitized, 100000); // Max 100k characters

    return sanitized;
  }

  sanitizeMessageContent(content: string, role: 'user' | 'assistant' | 'system'): string {
    let sanitized = this.sanitizeUserInput(content);

    // Role-specific sanitization
    switch (role) {
      case 'user':
        // User messages: strict sanitization
        sanitized = this.stripAllHTML(sanitized);
        break;

      case 'assistant':
        // Assistant messages: allow safe markdown
        sanitized = this.sanitizeMarkdown(sanitized);
        break;

      case 'system':
        // System messages: very strict
        sanitized = this.stripAllHTML(sanitized);
        sanitized = this.validateSystemPrompt(sanitized);
        break;
    }

    return sanitized;
  }

  private removeExcessiveWhitespace(text: string): string {
    return text
      .replace(/\s+/g, ' ') // Multiple whitespace to single space
      .replace(/\n\s*\n\s*\n/g, '\n\n') // Multiple newlines to double newline
      .trim();
  }

  private limitLength(text: string, maxLength: number): string {
    if (text.length <= maxLength) {
      return text;
    }

    return text.substring(0, maxLength - 3) + '...';
  }

  private stripAllHTML(text: string): string {
    return text.replace(/<[^>]*>/g, '');
  }

  private sanitizeMarkdown(text: string): string {
    // Allow safe markdown while preventing XSS
    return this.sanitizer.sanitize(text, {
      ALLOWED_TAGS: [
        'p', 'br', 'strong', 'em', 'code', 'pre',
        'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
        'ul', 'ol', 'li', 'blockquote'
      ],
      FORBID_SCRIPT: true,
      STRIP_COMMENTS: true
    });
  }

  private validateSystemPrompt(prompt: string): string {
    // System prompts should not contain certain patterns
    const forbiddenPatterns = [
      /javascript:/i,
      /data:/i,
      /vbscript:/i,
      /<script/i,
      /eval\(/i,
      /function\s*\(/i
    ];

    for (const pattern of forbiddenPatterns) {
      if (pattern.test(prompt)) {
        throw new Error('System prompt contains forbidden content');
      }
    }

    return prompt;
  }
}
```

### 8.3 Access Control & Authorization

#### Permission-Based Access Control

```typescript
interface Permission {
  resource: string;
  action: 'create' | 'read' | 'update' | 'delete';
  conditions?: Record<string, any>;
}

interface Role {
  name: string;
  permissions: Permission[];
}

class AccessControlService {
  private roles: Map<string, Role> = new Map();

  constructor() {
    this.initializeRoles();
  }

  private initializeRoles(): void {
    // User role - standard user permissions
    this.roles.set('user', {
      name: 'user',
      permissions: [
        { resource: 'threads', action: 'create', conditions: { ownerId: 'self' } },
        { resource: 'threads', action: 'read', conditions: { ownerId: 'self' } },
        { resource: 'threads', action: 'update', conditions: { ownerId: 'self' } },
        { resource: 'threads', action: 'delete', conditions: { ownerId: 'self' } },

        { resource: 'messages', action: 'create', conditions: { threadOwner: 'self' } },
        { resource: 'messages', action: 'read', conditions: { threadOwner: 'self' } },

        { resource: 'profile', action: 'read', conditions: { userId: 'self' } },
        { resource: 'profile', action: 'update', conditions: { userId: 'self' } },

        { resource: 'models', action: 'read' },
        { resource: 'health', action: 'read' }
      ]
    });

    // Admin role - elevated permissions
    this.roles.set('admin', {
      name: 'admin',
      permissions: [
        { resource: '*', action: 'create' },
        { resource: '*', action: 'read' },
        { resource: '*', action: 'update' },
        { resource: '*', action: 'delete' },

        { resource: 'users', action: 'create' },
        { resource: 'users', action: 'update' },
        { resource: 'users', action: 'delete' },

        { resource: 'system', action: 'read' },
        { resource: 'system', action: 'update' }
      ]
    });
  }

  async checkPermission(
    userId: string,
    resource: string,
    action: string,
    context: Record<string, any> = {}
  ): Promise<boolean> {
    const userRoles = await this.getUserRoles(userId);

    for (const roleName of userRoles) {
      const role = this.roles.get(roleName);
      if (!role) continue;

      for (const permission of role.permissions) {
        if (this.matchesPermission(permission, resource, action, userId, context)) {
          return true;
        }
      }
    }

    return false;
  }

  private matchesPermission(
    permission: Permission,
    resource: string,
    action: string,
    userId: string,
    context: Record<string, any>
  ): boolean {
    // Check resource match (wildcard support)
    if (permission.resource !== '*' && permission.resource !== resource) {
      return false;
    }

    // Check action match
    if (permission.action !== action) {
      return false;
    }

    // Check conditions
    if (permission.conditions) {
      for (const [key, value] of Object.entries(permission.conditions)) {
        if (value === 'self') {
          // Special case: user can only access their own resources
          const contextUserId = context[key] || context.userId;
          if (contextUserId !== userId) {
            return false;
          }
        } else if (context[key] !== value) {
          return false;
        }
      }
    }

    return true;
  }

  private async getUserRoles(userId: string): Promise<string[]> {
    // For now, all users have 'user' role
    // Admin users would be configured separately
    const adminUsers = (process.env.ADMIN_USERS || '').split(',');

    if (adminUsers.includes(userId)) {
      return ['user', 'admin'];
    }

    return ['user'];
  }
}

// Express middleware for authorization
function requirePermission(resource: string, action: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      const userId = req.user?.id;
      if (!userId) {
        return res.status(401).json({
          success: false,
          error: { code: 'UNAUTHORIZED', message: 'Authentication required' }
        });
      }

      const accessControl = new AccessControlService();
      const hasPermission = await accessControl.checkPermission(
        userId,
        resource,
        action,
        {
          ...req.params,
          ...req.query,
          userId: req.user.id
        }
      );

      if (!hasPermission) {
        return res.status(403).json({
          success: false,
          error: { code: 'FORBIDDEN', message: 'Insufficient permissions' }
        });
      }

      next();
    } catch (error) {
      next(error);
    }
  };
}
```

---

## 9. Backup & Recovery

### 9.1 Backup Strategy

#### Automated Backup System

```typescript
interface BackupConfig {
  frequency: 'hourly' | 'daily' | 'weekly';
  retention: {
    daily: number;
    weekly: number;
    monthly: number;
  };
  destinations: BackupDestination[];
  compression: boolean;
  encryption: boolean;
}

interface BackupDestination {
  type: 'gcs' | 's3' | 'local';
  path: string;
  credentials?: any;
}

class BackupService {
  constructor(
    private database: Database,
    private config: BackupConfig
  ) {}

  async createBackup(type: 'full' | 'incremental' = 'full'): Promise<BackupResult> {
    const backupId = this.generateBackupId();
    const timestamp = new Date().toISOString();

    try {
      const backupData = await this.exportData(type);

      // Compress if enabled
      let processedData = backupData;
      if (this.config.compression) {
        processedData = await this.compressData(backupData);
      }

      // Encrypt if enabled
      if (this.config.encryption) {
        processedData = await this.encryptData(processedData);
      }

      // Save to all configured destinations
      const destinations = await Promise.all(
        this.config.destinations.map(dest =>
          this.saveToDestination(dest, backupId, processedData)
        )
      );

      // Record backup metadata
      await this.recordBackupMetadata({
        id: backupId,
        type,
        timestamp,
        size: processedData.length,
        destinations,
        compressed: this.config.compression,
        encrypted: this.config.encryption,
        status: 'completed'
      });

      return {
        success: true,
        backupId,
        size: processedData.length,
        destinations: destinations.length
      };

    } catch (error) {
      await this.recordBackupMetadata({
        id: backupId,
        type,
        timestamp,
        status: 'failed',
        error: error.message
      });

      throw error;
    }
  }

  private async exportData(type: 'full' | 'incremental'): Promise<string> {
    const collections = ['users', 'threads', 'messages', 'models_config', 'operations', 'system'];
    const exportData: any = {
      version: '1.0',
      timestamp: Date.now(),
      type,
      collections: {}
    };

    for (const collection of collections) {
      if (type === 'incremental') {
        // Only export changed data since last backup
        const lastBackupTime = await this.getLastBackupTime();
        exportData.collections[collection] = await this.database
          .collection(collection)
          .where('updatedAt', '>', lastBackupTime)
          .get();
      } else {
        // Full export
        exportData.collections[collection] = await this.database
          .collection(collection)
          .get();
      }
    }

    return JSON.stringify(exportData, null, 2);
  }

  private async compressData(data: string): Promise<Buffer> {
    const zlib = require('zlib');
    return new Promise((resolve, reject) => {
      zlib.gzip(data, (err, compressed) => {
        if (err) reject(err);
        else resolve(compressed);
      });
    });
  }

  private async encryptData(data: Buffer | string): Promise<Buffer> {
    const crypto = require('crypto');
    const algorithm = 'aes-256-gcm';
    const key = Buffer.from(process.env.BACKUP_ENCRYPTION_KEY, 'hex');
    const iv = crypto.randomBytes(16);

    const cipher = crypto.createCipher(algorithm, key);

    let encrypted = cipher.update(data, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    const authTag = cipher.getAuthTag();

    // Combine IV, auth tag, and encrypted data
    return Buffer.concat([
      iv,
      authTag,
      Buffer.from(encrypted, 'hex')
    ]);
  }

  async restoreFromBackup(backupId: string): Promise<void> {
    try {
      const metadata = await this.getBackupMetadata(backupId);
      if (!metadata) {
        throw new Error(`Backup ${backupId} not found`);
      }

      // Download backup data
      const encryptedData = await this.downloadFromDestination(
        metadata.destinations[0],
        backupId
      );

      // Decrypt if necessary
      let processedData = encryptedData;
      if (metadata.encrypted) {
        processedData = await this.decryptData(encryptedData);
      }

      // Decompress if necessary
      if (metadata.compressed) {
        processedData = await this.decompressData(processedData);
      }

      // Parse backup data
      const backupData = JSON.parse(processedData.toString());

      // Restore each collection
      for (const [collection, documents] of Object.entries(backupData.collections)) {
        await this.restoreCollection(collection, documents as any[]);
      }

      console.log(`Successfully restored from backup ${backupId}`);

    } catch (error) {
      console.error(`Failed to restore from backup ${backupId}:`, error);
      throw error;
    }
  }

  private async restoreCollection(collection: string, documents: any[]): Promise<void> {
    // Clear existing data (with confirmation in production)
    await this.database.collection(collection).deleteAll();

    // Batch insert restored documents
    const batchSize = 100;
    for (let i = 0; i < documents.length; i += batchSize) {
      const batch = documents.slice(i, i + batchSize);
      await this.database.collection(collection).batchCreate(batch);
    }
  }
}
```

### 9.2 Point-in-Time Recovery

#### Transaction Log Based Recovery

```typescript
interface TransactionLog {
  id: string;
  timestamp: number;
  operation: 'create' | 'update' | 'delete';
  collection: string;
  documentId: string;
  beforeData?: any;
  afterData?: any;
  userId?: string;
  sessionId?: string;
}

class PointInTimeRecovery {
  constructor(
    private database: Database,
    private backupService: BackupService
  ) {}

  async recoverToPoint(targetTimestamp: number): Promise<void> {
    // Find the most recent backup before target time
    const baseBackup = await this.findBaseBackup(targetTimestamp);
    if (!baseBackup) {
      throw new Error('No suitable base backup found');
    }

    // Restore from base backup
    await this.backupService.restoreFromBackup(baseBackup.id);

    // Apply transaction logs from backup time to target time
    const transactionLogs = await this.getTransactionLogs(
      baseBackup.timestamp,
      targetTimestamp
    );

    await this.applyTransactionLogs(transactionLogs);

    console.log(`Recovery completed to ${new Date(targetTimestamp).toISOString()}`);
  }

  async logTransaction(
    operation: string,
    collection: string,
    documentId: string,
    beforeData?: any,
    afterData?: any,
    userId?: string
  ): Promise<void> {
    const logEntry: TransactionLog = {
      id: this.generateLogId(),
      timestamp: Date.now(),
      operation: operation as any,
      collection,
      documentId,
      beforeData,
      afterData,
      userId,
      sessionId: this.getCurrentSessionId()
    };

    // Store transaction log
    await this.database.create('transaction_logs', logEntry);

    // Cleanup old logs (retain for 30 days)
    await this.cleanupOldLogs();
  }

  private async findBaseBackup(targetTimestamp: number): Promise<any> {
    const backups = await this.database
      .collection('backup_metadata')
      .where('timestamp', '<=', targetTimestamp)
      .orderBy('timestamp', 'desc')
      .limit(1)
      .get();

    return backups[0] || null;
  }

  private async getTransactionLogs(fromTime: number, toTime: number): Promise<TransactionLog[]> {
    return this.database
      .collection('transaction_logs')
      .where('timestamp', '>', fromTime)
      .where('timestamp', '<=', toTime)
      .orderBy('timestamp', 'asc')
      .get();
  }

  private async applyTransactionLogs(logs: TransactionLog[]): Promise<void> {
    for (const log of logs) {
      try {
        switch (log.operation) {
          case 'create':
            if (log.afterData) {
              await this.database.create(log.collection, log.afterData);
            }
            break;

          case 'update':
            if (log.afterData) {
              await this.database.update(log.collection, log.documentId, log.afterData);
            }
            break;

          case 'delete':
            await this.database.delete(log.collection, log.documentId);
            break;
        }
      } catch (error) {
        console.warn(`Failed to apply transaction log ${log.id}:`, error);
        // Continue with recovery - some operations might fail due to constraints
      }
    }
  }

  private async cleanupOldLogs(): Promise<void> {
    const cutoffTime = Date.now() - (30 * 24 * 60 * 60 * 1000); // 30 days ago

    await this.database
      .collection('transaction_logs')
      .where('timestamp', '<', cutoffTime)
      .delete();
  }
}
```

---

## 10. Database Provider Abstraction

### 10.1 Unified Database Interface

```typescript
interface DatabaseProvider {
  // Connection management
  connect(config: DatabaseConfig): Promise<void>;
  disconnect(): Promise<void>;
  isConnected(): boolean;

  // Document operations
  create(collection: string, data: any): Promise<string>;
  get(collection: string, id: string): Promise<any>;
  update(collection: string, id: string, updates: any): Promise<void>;
  delete(collection: string, id: string): Promise<void>;

  // Query operations
  find(collection: string, query: QueryBuilder): Promise<any[]>;
  findOne(collection: string, query: QueryBuilder): Promise<any>;
  count(collection: string, query: QueryBuilder): Promise<number>;

  // Batch operations
  batchCreate(collection: string, documents: any[]): Promise<string[]>;
  batchUpdate(collection: string, updates: BatchUpdate[]): Promise<void>;
  batchDelete(collection: string, ids: string[]): Promise<void>;

  // Real-time features
  watch(collection: string, query: QueryBuilder, callback: WatchCallback): Promise<WatchHandle>;
  unwatch(handle: WatchHandle): Promise<void>;

  // Transactions
  startTransaction(): Promise<Transaction>;
  commitTransaction(transaction: Transaction): Promise<void>;
  rollbackTransaction(transaction: Transaction): Promise<void>;
}

interface QueryBuilder {
  where(field: string, operator: string, value: any): QueryBuilder;
  orderBy(field: string, direction: 'asc' | 'desc'): QueryBuilder;
  limit(count: number): QueryBuilder;
  offset(count: number): QueryBuilder;
  startAfter(cursor: any): QueryBuilder;
  build(): any;
}

interface WatchCallback {
  (change: DatabaseChange): void;
}

interface DatabaseChange {
  type: 'added' | 'modified' | 'removed';
  document: any;
  oldDocument?: any;
}
```

### 10.2 Firebase Implementation

```typescript
class FirebaseProvider implements DatabaseProvider {
  private database: any;
  private admin: any;

  constructor() {
    this.admin = require('firebase-admin');
  }

  async connect(config: DatabaseConfig): Promise<void> {
    if (config.emulator) {
      // Connect to Firebase Emulator for development
      process.env['FIREBASE_DATABASE_EMULATOR_HOST'] = config.emulator.host;
      process.env['GCLOUD_PROJECT'] = config.projectId;
    }

    this.admin.initializeApp({
      credential: config.serviceAccount
        ? this.admin.credential.cert(config.serviceAccount)
        : this.admin.credential.applicationDefault(),
      databaseURL: config.databaseURL
    });

    this.database = this.admin.database();
  }

  async create(collection: string, data: any): Promise<string> {
    const ref = this.database.ref(collection).push();
    await ref.set({
      ...data,
      id: ref.key,
      createdAt: this.admin.database.ServerValue.TIMESTAMP,
      updatedAt: this.admin.database.ServerValue.TIMESTAMP
    });
    return ref.key;
  }

  async get(collection: string, id: string): Promise<any> {
    const snapshot = await this.database.ref(`${collection}/${id}`).once('value');
    return snapshot.val();
  }

  async update(collection: string, id: string, updates: any): Promise<void> {
    await this.database.ref(`${collection}/${id}`).update({
      ...updates,
      updatedAt: this.admin.database.ServerValue.TIMESTAMP
    });
  }

  async delete(collection: string, id: string): Promise<void> {
    await this.database.ref(`${collection}/${id}`).remove();
  }

  async find(collection: string, query: QueryBuilder): Promise<any[]> {
    const builtQuery = query.build();
    let ref = this.database.ref(collection);

    // Apply Firebase-specific query constraints
    if (builtQuery.where) {
      for (const condition of builtQuery.where) {
        ref = ref.orderByChild(condition.field);
        if (condition.operator === '==') {
          ref = ref.equalTo(condition.value);
        } else if (condition.operator === '>') {
          ref = ref.startAt(condition.value);
        } else if (condition.operator === '<') {
          ref = ref.endAt(condition.value);
        }
      }
    }

    if (builtQuery.limit) {
      ref = ref.limitToLast(builtQuery.limit);
    }

    const snapshot = await ref.once('value');
    const data = snapshot.val();

    return data ? Object.values(data) : [];
  }

  async watch(collection: string, query: QueryBuilder, callback: WatchCallback): Promise<WatchHandle> {
    const builtQuery = query.build();
    let ref = this.database.ref(collection);

    // Apply query constraints (similar to find method)
    // ... query building logic ...

    const listener = ref.on('child_changed', (snapshot) => {
      callback({
        type: 'modified',
        document: snapshot.val()
      });
    });

    // Return handle for cleanup
    return {
      collection,
      ref,
      listener,
      unsubscribe: () => ref.off('child_changed', listener)
    };
  }

  async unwatch(handle: WatchHandle): Promise<void> {
    handle.unsubscribe();
  }
}
```

### 10.3 MongoDB Implementation

```typescript
class MongoProvider implements DatabaseProvider {
  private client: any;
  private db: any;

  async connect(config: DatabaseConfig): Promise<void> {
    const { MongoClient } = require('mongodb');

    this.client = new MongoClient(config.connectionString, {
      useUnifiedTopology: true
    });

    await this.client.connect();
    this.db = this.client.db(config.databaseName);
  }

  async create(collection: string, data: any): Promise<string> {
    const doc = {
      ...data,
      _id: new (require('mongodb')).ObjectId(),
      createdAt: new Date(),
      updatedAt: new Date()
    };

    await this.db.collection(collection).insertOne(doc);
    return doc._id.toString();
  }

  async get(collection: string, id: string): Promise<any> {
    const ObjectId = require('mongodb').ObjectId;
    const doc = await this.db.collection(collection).findOne({ _id: new ObjectId(id) });

    if (doc) {
      doc.id = doc._id.toString();
      delete doc._id;
    }

    return doc;
  }

  async update(collection: string, id: string, updates: any): Promise<void> {
    const ObjectId = require('mongodb').ObjectId;

    await this.db.collection(collection).updateOne(
      { _id: new ObjectId(id) },
      {
        $set: {
          ...updates,
          updatedAt: new Date()
        }
      }
    );
  }

  async find(collection: string, query: QueryBuilder): Promise<any[]> {
    const builtQuery = query.build();
    let cursor = this.db.collection(collection).find(builtQuery.filter || {});

    if (builtQuery.sort) {
      cursor = cursor.sort(builtQuery.sort);
    }

    if (builtQuery.limit) {
      cursor = cursor.limit(builtQuery.limit);
    }

    if (builtQuery.skip) {
      cursor = cursor.skip(builtQuery.skip);
    }

    const docs = await cursor.toArray();

    // Transform _id to id
    return docs.map(doc => ({
      ...doc,
      id: doc._id.toString(),
      _id: undefined
    }));
  }

  async watch(collection: string, query: QueryBuilder, callback: WatchCallback): Promise<WatchHandle> {
    const changeStream = this.db.collection(collection).watch();

    changeStream.on('change', (change) => {
      let type: 'added' | 'modified' | 'removed';

      switch (change.operationType) {
        case 'insert':
          type = 'added';
          break;
        case 'update':
        case 'replace':
          type = 'modified';
          break;
        case 'delete':
          type = 'removed';
          break;
        default:
          return;
      }

      callback({
        type,
        document: change.fullDocument,
        oldDocument: change.fullDocumentBeforeChange
      });
    });

    return {
      collection,
      changeStream,
      unsubscribe: () => changeStream.close()
    };
  }
}
```

### 10.4 Database Factory & Configuration

```typescript
class DatabaseFactory {
  static create(provider: 'firebase' | 'mongodb', config: DatabaseConfig): DatabaseProvider {
    switch (provider) {
      case 'firebase':
        return new FirebaseProvider();
      case 'mongodb':
        return new MongoProvider();
      default:
        throw new Error(`Unsupported database provider: ${provider}`);
    }
  }
}

// Configuration management
class DatabaseConfigManager {
  static getConfig(): { provider: string; config: DatabaseConfig } {
    const provider = process.env.DB_PROVIDER || 'firebase';

    switch (provider) {
      case 'firebase':
        return {
          provider,
          config: {
            projectId: process.env.FIREBASE_PROJECT_ID,
            databaseURL: process.env.FIREBASE_DATABASE_URL,
            serviceAccount: process.env.FIREBASE_SERVICE_ACCOUNT
              ? JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT)
              : null,
            emulator: process.env.NODE_ENV === 'development' ? {
              host: 'localhost:9000'
            } : null
          }
        };

      case 'mongodb':
        return {
          provider,
          config: {
            connectionString: process.env.MONGODB_CONNECTION_STRING,
            databaseName: process.env.MONGODB_DATABASE_NAME
          }
        };

      default:
        throw new Error(`Unsupported database provider: ${provider}`);
    }
  }
}

// Usage in application
export async function initializeDatabase(): Promise<DatabaseProvider> {
  const { provider, config } = DatabaseConfigManager.getConfig();
  const database = DatabaseFactory.create(provider as any, config);

  await database.connect(config);

  return database;
}
```

---

This comprehensive data model and persistence documentation provides a complete foundation for implementing a robust, scalable, and secure database layer for the LLM Chat application. The modular design supports multiple database providers while maintaining consistency and enabling smooth migration between systems as requirements evolve.