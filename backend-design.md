# Backend Architecture & Implementation

The backend follows a **layered service architecture** built on Node.js with Express, designed for scalability, maintainability, and integration with multiple LLM providers. The architecture emphasizes clean separation of concerns, robust error handling, and real-time communication capabilities.

## 1. Core Architecture Principles

- **Service-oriented design**: Each major feature area is encapsulated in dedicated service classes
- **Provider abstraction**: Unified interface for different LLM providers (Claude, OpenAI, Google AI, Ollama)
- **Streaming-first**: Built-in support for real-time streaming responses via Server-Sent Events
- **Security by design**: End-to-end encryption for API keys, JWT-based authentication, comprehensive input validation
- **Observability**: Structured logging, health checks, and metrics collection at every layer

## 2. Request Processing Pipeline

```
Client Request → Auth Middleware → Validation → Service Layer → Provider Layer → Database Layer
                     ↓              ↓             ↓              ↓              ↓
               JWT Verification → Schema Check → Business Logic → Model Call → Data Persistence
                     ↓              ↓             ↓              ↓              ↓
               Rate Limiting → Error Handling → Response Format → Streaming → Audit Log
```

## 3. Core Services & Components

### Authentication Service (`AuthService`)

- **JWT Management**: Token generation, validation, and refresh logic
- **Session Handling**: httpOnly cookie management with secure flags
- **Password Security**: bcrypt hashing with configurable salt rounds
- **Rate Limiting**: Login attempt throttling per IP and user

### Thread Management Service (`ThreadService`)

- **CRUD Operations**: Create, read, update, delete threads with proper authorization
- **Search Functionality**: Full-text search across thread titles and summaries
- **Pagination**: Cursor-based pagination for efficient large dataset handling
- **Model Context Tracking**: Separate conversation contexts per model within threads

### Message Processing Service (`MessageService`)

- **Message Storage**: Efficient storage and retrieval with indexing
- **Context Assembly**: Builds conversation context from system prompts, summaries, and recent messages
- **Token Management**: Tracks and limits context tokens per model requirements
- **Streaming Coordination**: Manages real-time message delivery via SSE

### Model Proxy Service (`ModelProxyService`)

- **Provider Abstraction**: Unified interface for all LLM providers
- **Request Routing**: Intelligently routes requests to appropriate providers
- **Response Normalization**: Standardizes responses across different provider formats
- **Error Handling**: Provider-specific error translation and retry logic

### Summarization Service (`SummarizationService`)

- **Configurable Triggers**: Auto-summarization after N messages (configurable per user, default: 10)
- **Async Processing**: Non-blocking summarization triggered per conversation completion
- **Context-Aware**: Uses conversation history to generate relevant summaries
- **Token Optimization**: Maintains summaries within 300-700 token range
- **Same Model Usage**: Uses the same model that was used in the conversation for summarization
- **Fallback Handling**: Graceful degradation when summarization fails
- **Job Tracking**: Tracks summarization job status (pending/generating/complete/failed)

### Health Monitoring Service (`HealthService`)

- **System Status**: Real-time monitoring of all system components
- **Database Connectivity**: Continuous database health checks
- **Model Availability**: Tracks loading status of local and remote models
- **Performance Metrics**: Response times, error rates, and resource utilization

## 4. Authentication & Authorization

### JWT Implementation

#### Token Structure
```json
{
  "sub": "user_id",
  "username": "user@example.com",
  "iat": 1705401600000,
  "exp": 1705405200000,
  "scope": ["read", "write"]
}
```

#### Security Features
- HS256 signing with rotating secrets
- Short-lived tokens (15 minutes) with refresh capability
- httpOnly cookies with secure and sameSite flags
- Automatic token refresh before expiration

### Authorization Middleware

- **Route Protection**: Middleware validates JWT on protected endpoints
- **Role-Based Access**: Future-ready for admin/user role separation
- **API Key Validation**: Verifies user has required API keys for model access
- **Rate Limiting**: Per-user request throttling to prevent abuse

## 5. Model Integration Layer

### Unified Model Interface

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

### Provider Implementations

- **Claude Provider**: Anthropic API integration with streaming support
- **OpenAI Provider**: GPT model access with function calling capabilities
- **Google AI Provider**: Gemini model integration
- **Ollama Provider**: Local model management with loading orchestration

### Model Loading Orchestration

- **On-Demand Loading**: Local models loaded only when requested
- **Status Tracking**: Real-time loading progress via SSE or polling
- **Resource Management**: Memory and CPU optimization for model operations
- **Failure Recovery**: Automatic retry logic with exponential backoff

## 6. Real-time Communication

### Server-Sent Events (SSE) Implementation

- **Connection Management**: Maintains persistent connections for streaming
- **Message Formatting**: Structured SSE messages with proper event types
- **Error Recovery**: Automatic reconnection with exponential backoff
- **Heartbeat**: Keep-alive messages to prevent connection timeouts

### Database Watchers and Timeout Management

#### Watcher Configuration
- **Message Generation Timeout**: 30 seconds maximum per message
- **Model Loading Timeout**: 300 seconds (5 minutes) for local model loading
- **Health Check Timeout**: 5 seconds for external service checks
- **Summary Generation Timeout**: 60 seconds for async summarization

#### Watcher Implementation
- **Firebase**: Real-time listeners with automatic cleanup on disconnect
- **MongoDB**: Change streams with configurable batch size
- **Polling Fallback**: 100ms intervals when real-time watching unavailable

#### Resource Limits
- **Max Concurrent Watchers**: 100 per backend instance
- **Memory Limit**: 50MB total for all active watchers
- **Connection Cleanup**: Automatic cleanup after client disconnect detection

#### Error Handling
- **Timeout Events**: Sent via SSE before connection closure
- **Watcher Failures**: Automatic fallback to polling mode
- **Resource Exhaustion**: Queue new watchers when limit reached
- **DB Connection Loss**: Graceful degradation with cached responses

### Streaming Architecture

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

## 7. Performance Optimization

### Caching Strategy (Redis-free Initial Implementation)

- **In-Memory Caching**: LRU cache for frequently accessed data
- **Model Status Caching**: Cache model availability to reduce API calls
- **Thread Metadata Caching**: Cache thread summaries and metadata
- **API Response Caching**: Cache non-sensitive API responses

### Connection Management

- **HTTP Keep-Alive**: Reuse connections to external APIs
- **Connection Pooling**: Efficient database connection management
- **Request Batching**: Batch non-critical operations where possible

## 8. Security Implementation

### API Key Management

- **Encryption at Rest**: AES-256-GCM encryption with KMS-managed keys
- **Secure Transmission**: TLS 1.3 for all external communications
- **Key Rotation**: Support for API key updates without service interruption
- **Access Logging**: Audit trail for all API key usage

### Input Validation & Sanitization

- **Schema Validation**: Joi/Zod schemas for all request payloads
- **Content Filtering**: Sanitize user inputs to prevent injection attacks
- **File Upload Security**: Validate file types and scan for malicious content
- **Rate Limiting**: Configurable limits per user and endpoint

## 9. Error Handling & Logging

### Structured Error Response Format

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
    "timestamp": 1705401800000
  }
}
```

### Logging Strategy

- **Structured Logging**: JSON format with correlation IDs
- **Log Levels**: DEBUG, INFO, WARN, ERROR with appropriate filtering
- **Sensitive Data Redaction**: Automatic redaction of API keys and personal data
- **Request Tracing**: Full request lifecycle tracking for debugging

# 10. API Endpoints (Comprehensive Specification)

All API endpoints use RESTful conventions with JSON request/response bodies. Authentication is handled via JWT tokens in httpOnly cookies. Rate limiting is applied per user and endpoint.

## 10.1. Standard Response Format

### Success Response
```json
{
  "success": true,
  "data": { /* endpoint-specific data */ },
  "pagination": { /* for list endpoints */ },
  "timestamp": 1705401800000,
  "requestId": "req_123456789"
}
```

### Error Response
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": { /* optional error details */ }
  },
  "timestamp": 1705401800000,
  "requestId": "req_123456789"
}
```

### Pagination Format (for list endpoints)
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

## 10.2. Authentication Endpoints

### POST /api/login

**Description**: Authenticate user and create session
**Authentication**: Public
**Rate Limit**: 5 requests per minute per IP

#### Request
```json
{
  "username": "user@example.com",
  "password": "securepassword123"
}
```

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user_123",
      "username": "user@example.com",
      "displayName": "John Doe",
      "createdAt": 1704067200000
    },
    "expiresIn": 900
  }
}
```
**Note**: JWT token set as httpOnly, secure, sameSite cookie

#### Error Responses
- `401 INVALID_CREDENTIALS`: Invalid username or password
- `429 RATE_LIMITED`: Too many login attempts
- `400 VALIDATION_ERROR`: Missing or invalid request fields

### POST /api/logout

**Description**: Clear authentication session
**Authentication**: Optional
**Rate Limit**: None

**Request Body**: Empty

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "message": "Successfully logged out"
  }
}
```

### GET /api/health

**Description**: System health check
**Authentication**: Public
**Rate Limit**: 10 requests per minute per IP

#### Success Response (200)
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

### GET /api/version

**Description**: Get application version information
**Authentication**: Required
**Rate Limit**: 30 requests per minute per user

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "version": "1.2.3",
    "buildTime": 1705305600000,
    "commit": "abc123def456",
    "environment": "production"
  }
}
```

## 10.3. Profile Management Endpoints

### GET /api/profile

**Description**: Get user profile information
**Authentication**: Required
**Rate Limit**: 60 requests per minute per user

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user_123",
      "username": "user@example.com",
      "displayName": "John Doe",
      "createdAt": 1704067200000,
      "settings": {
        "defaultModel": "claude-opus",
        "defaultTemperature": 0.7,
        "systemPrompt": "You are a helpful assistant...",
        "summarizationInterval": 10
      },
      "apiKeys": {
        "claude": {
          "configured": true,
          "lastUsed": 1705232200000
        },
        "openai": {
          "configured": false
        },
        "google": {
          "configured": true,
          "lastUsed": 1705154400000
        }
      }
    }
  }
}
```

### PATCH /api/profile

**Description**: Update user profile settings
**Authentication**: Required
**Rate Limit**: 20 requests per minute per user

#### Request
```json
{
  "displayName": "John Smith",
  "settings": {
    "defaultModel": "gpt-4",
    "defaultTemperature": 0.8,
    "systemPrompt": "You are an expert coding assistant...",
    "summarizationInterval": 15
  }
}
```

#### Success Response (200)
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
        "systemPrompt": "You are an expert coding assistant...",
        "summarizationInterval": 15
      }
    }
  }
}
```

### POST /api/profile/api-keys

**Description**: Add or update API keys for external providers
**Authentication**: Required
**Rate Limit**: 5 requests per minute per user

#### Request
```json
{
  "claude": "sk-ant-api03-...",
  "openai": "sk-proj-...",
  "google": "AIza..."
}
```

#### Success Response (200)
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

## 10.4. Model Management Endpoints

### GET /api/models

**Description**: Get available model catalog
**Authentication**: Required
**Rate Limit**: 60 requests per minute per user

#### Success Response (200)
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

### GET /api/models/:modelId/status

**Description**: Get specific model status and loading information
**Authentication**: Required
**Rate Limit**: 30 requests per minute per user

**URL Parameters:**
- `modelId`: Model identifier (e.g., "gemma-2b-local")

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "modelId": "gemma-2b-local",
    "status": "loaded",
    "loadProgress": 100,
    "memoryUsage": "2.1GB",
    "lastLoaded": 1705310200000,
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

### POST /api/models/:modelId/load

**Description**: Load a local model (Ollama)
**Authentication**: Required
**Rate Limit**: 3 requests per minute per user

**URL Parameters:**
- `modelId`: Model identifier (e.g., "gemma-2b-local")

**Request Body**: Empty

#### Success Response (202)
```json
{
  "success": true,
  "data": {
    "modelId": "gemma-2b-local",
    "status": "loading"
  }
}
```

## 10.5. Thread Management Endpoints

### GET /api/threads

**Description**: List user's threads with pagination and search
**Authentication**: Required
**Rate Limit**: 100 requests per minute per user

#### Query Parameters
- `limit` (optional): Number of items per page (default: 20, max: 100)
- `cursor` (optional): Pagination cursor for next page
- `q` (optional): Search query (searches title and summary)
- `orderBy` (optional): Sort order ("created_at", "updated_at") default: "updated_at"
- `order` (optional): Sort direction ("asc", "desc") default: "desc"

#### Example Request
```
GET /api/threads?limit=20&q=architecture&orderBy=updated_at&order=desc
```

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "threads": [
      {
        "id": "thread_123",
        "title": "System Architecture Discussion",
        "createdAt": 1705230000000,
        "updatedAt": 1705319400000,
        "messageCount": 15,
        "activeModels": ["claude-opus", "gpt-4"],
        "lastMessage": {
          "content": "That's a great approach for microservices...",
          "role": "assistant",
          "model": "claude-opus",
          "timestamp": 1705401800000
        }
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

### POST /api/threads

**Description**: Create a new thread with first message and initiate conversation
**Authentication**: Required
**Rate Limit**: 10 requests per minute per user

#### Request
```json
{
  "title": "New Discussion Topic",
  "firstMessage": {
    "content": "Help me understand microservices architecture",
    "modelId": "claude-opus",
    "temperature": 0.7
  }
}
```

#### Success Response (201)
```json
{
  "success": true,
  "data": {
    "threadId": "thread_456",
    "title": "Microservices Architecture Help",
    "userMessageId": "msg_001",
    "assistantMessageId": "msg_002",
    "modelId": "claude-opus",
    "timestamp": 1705320300000
  }
}
```

**Behavior:**
1. Creates new thread in database
2. Stores user's first message with `userMessageId`
3. Creates assistant message placeholder with `assistantMessageId`
4. **ASYNC**: Initiates LLM API call to generate response (fire-and-forget)
   - Background consumer stores response tokens in DB array as they stream in
   - Marks message status as 'complete' when response is fully received
5. **SYNC**: Calls local Ollama to generate 3-word title from first message content
6. Returns with thread ID, generated title, and message IDs
7. Client should use `GET /api/threads/:threadId/models/:modelId/messages/:messageId` to stream the response

**Optional Fields:**
- `title`: If not provided, will be auto-generated from first message content
- `temperature`: If not provided, uses user's default temperature setting

### GET /api/threads/:threadId

**Description**: Get thread metadata and summary
**Authentication**: Required (thread owner only)
**Rate Limit**: 60 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier

#### Success Response (200)
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
          "lastMessageAt": 1705319400000,
          "summary": "Discussion focused on microservice patterns...",
          "summaryUpdatedAt": 1705319100000
        },
        "gpt-4": {
          "messageCount": 7,
          "lastMessageAt": 1705316700000,
          "summary": "Comparison of architectural approaches...",
          "summaryUpdatedAt": 1705316400000
        }
      }
    }
  }
}
```

### DELETE /api/threads/:threadId

**Description**: Delete a thread and all its messages
**Authentication**: Required (thread owner only)
**Rate Limit**: 5 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "deletedThread": "thread_123",
    "deletedMessages": 15
  }
}
```

### PATCH /api/threads/:threadId/title

**Description**: Update thread title
**Authentication**: Required (thread owner only)
**Rate Limit**: 20 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier

#### Request
```json
{
  "title": "Updated Thread Title"
}
```

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "thread": {
      "id": "thread_123",
      "title": "Updated Thread Title",
      "updatedAt": 1705320600000
    }
  }
}
```

## 10.6. Message Management Endpoints

### GET /api/threads/:threadId/models/:modelId/messages

**Description**: Get messages for specific model in thread
**Authentication**: Required (thread owner only)
**Rate Limit**: 100 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier
- `modelId`: Model identifier

#### Query Parameters
- `limit` (optional): Number of messages (default: 50, max: 200)
- `cursor` (optional): Pagination cursor
- `order` (optional): "asc" or "desc" (default: "desc")

#### Success Response (200)
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
        "createdAt": 1705318800000,
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
        "createdAt": 1705318860000,
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

### POST /api/threads/:threadId/models/:modelId/messages

**Description**: Send message to model and initiate async response generation
**Authentication**: Required (thread owner only)
**Rate Limit**: 20 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier
- `modelId`: Model identifier

#### Request
```json
{
  "content": "Explain the benefits of event-driven architecture",
  "temperature": 0.8
}
```

#### Response (201)
```json
{
  "success": true,
  "data": {
    "userMessageId": "msg_125",
    "assistantMessageId": "msg_126",
    "threadId": "thread_123",
    "modelId": "claude-opus",
    "timestamp": 1705319700000
  }
}
```

**Behavior:**
1. Creates and stores user message immediately with `userMessageId`
2. Creates assistant message placeholder with `assistantMessageId`
3. **ASYNC** (single flow, fire-and-forget):
   - Fetches user system prompt
   - Fetches thread summary for the model
   - Fetches recent messages to build context
   - Applies token budget constraints
   - Calls LLM API with assembled context
   - Background consumer stores response tokens in DB array as they stream in
   - Marks message status as 'complete' when response is fully received
4. Returns immediately with message IDs for client to track
5. Client should use `GET /api/threads/:threadId/models/:modelId/messages/:messageId` to stream the response

### GET /api/threads/:threadId/models/:modelId/messages/:messageId

**Description**: Stream LLM response for a specific message (SSE endpoint)
**Authentication**: Required (thread owner only)
**Rate Limit**: 30 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier
- `modelId`: Model identifier
- `messageId`: Assistant message identifier

#### Streaming Response (SSE)
```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: token
data: {"token": "Event", "messageId": "msg_126", "index": 0}

event: token
data: {"token": "-driven", "messageId": "msg_126", "index": 1}

event: token
data: {"token": " architecture", "messageId": "msg_126", "index": 2}

event: message_complete
data: {"messageId": "msg_126", "totalTokens": 234, "timestamp": 1705319730000, "status": "complete"}

event: close
data: {}
```

**Behavior:**
1. Returns any existing tokens stored in DB for the `messageId` immediately
2. If message is not complete (status != 'complete'), registers DB watcher for updates
3. Streams new tokens as they are added to the message document array
4. **Deregisters DB watcher** when:
   - Message status becomes 'complete' (all tokens received)
   - Client closes SSE connection
   - 30-second timeout occurs
   - Error occurs during streaming
5. Closes SSE connection after watcher deregistration

**DB Watcher Configuration:**
- **Timeout Duration**: 30 seconds for message generation
- **Polling Interval**: Real-time with Firebase listeners (or 100ms polling for other DBs)
- **Max Active Watchers**: 100 per instance (to prevent resource exhaustion)
- **Cleanup Strategy**: Explicit deregistration on completion, disconnect, timeout, or error
- **Resource Management**: Automatic cleanup prevents memory leaks and zombie watchers

#### Error Responses
- `404 NOT_FOUND`: Message ID not found
- `410 GONE`: Message generation failed or timed out
- `429 TOO_MANY_REQUESTS`: Rate limit exceeded

### POST /api/threads/:threadId/models/:modelId/summarize

**Description**: Manually trigger summarization for thread/model
**Authentication**: Required (thread owner only)
**Rate Limit**: 5 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier
- `modelId`: Model identifier

#### Success Response (200)
```json
{
  "success": true,
  "data": {
    "summary": "This conversation explored event-driven architecture patterns, discussing benefits like loose coupling, scalability, and resilience. Key topics included message queues, event sourcing, and CQRS patterns.",
    "tokens": 45,
    "generatedAt": 1705320000000,
    "modelUsed": "claude-opus"
  }
}
```

## 10.7. Error Handling

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

## 10.8. Rate Limiting Headers

All responses include rate limiting headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642234567
X-RateLimit-Window: 60
```