# Backend Architecture & Implementation

The backend follows a **layered service architecture** built on Node.js with Express, designed for scalability, maintainability, and integration with multiple LLM providers. The architecture emphasizes clean separation of concerns, robust error handling, and real-time communication capabilities.

## Core Architecture Principles

- **Service-oriented design**: Each major feature area is encapsulated in dedicated service classes
- **Provider abstraction**: Unified interface for different LLM providers (Claude, OpenAI, Google AI, Ollama)
- **Streaming-first**: Built-in support for real-time streaming responses via Server-Sent Events
- **Security by design**: End-to-end encryption for API keys, JWT-based authentication, comprehensive input validation
- **Observability**: Structured logging, health checks, and metrics collection at every layer

## Request Processing Pipeline

```
Client Request → Auth Middleware → Validation → Service Layer → Provider Layer → Database Layer
                     ↓              ↓             ↓              ↓              ↓
               JWT Verification → Schema Check → Business Logic → Model Call → Data Persistence
                     ↓              ↓             ↓              ↓              ↓
               Rate Limiting → Error Handling → Response Format → Streaming → Audit Log
```

## Core Services & Components

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

- **Async Processing**: Non-blocking summarization triggered per conversation
- **Context-Aware**: Uses conversation history to generate relevant summaries
- **Token Optimization**: Maintains summaries within 300-700 token range
- **Fallback Handling**: Graceful degradation when summarization fails

### Health Monitoring Service (`HealthService`)

- **System Status**: Real-time monitoring of all system components
- **Database Connectivity**: Continuous database health checks
- **Model Availability**: Tracks loading status of local and remote models
- **Performance Metrics**: Response times, error rates, and resource utilization

## Authentication & Authorization

### JWT Implementation

#### Token Structure
```json
{
  "sub": "user_id",
  "username": "user@example.com",
  "iat": 1234567890,
  "exp": 1234567890,
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

## Model Integration Layer

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

## Real-time Communication

### Server-Sent Events (SSE) Implementation

- **Connection Management**: Maintains persistent connections for streaming
- **Message Formatting**: Structured SSE messages with proper event types
- **Error Recovery**: Automatic reconnection with exponential backoff
- **Heartbeat**: Keep-alive messages to prevent connection timeouts

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

## Performance Optimization

### Caching Strategy (Redis-free Initial Implementation)

- **In-Memory Caching**: LRU cache for frequently accessed data
- **Model Status Caching**: Cache model availability to reduce API calls
- **Thread Metadata Caching**: Cache thread summaries and metadata
- **API Response Caching**: Cache non-sensitive API responses

### Connection Management

- **HTTP Keep-Alive**: Reuse connections to external APIs
- **Connection Pooling**: Efficient database connection management
- **Request Batching**: Batch non-critical operations where possible

## Security Implementation

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

## Error Handling & Logging

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
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Logging Strategy

- **Structured Logging**: JSON format with correlation IDs
- **Log Levels**: DEBUG, INFO, WARN, ERROR with appropriate filtering
- **Sensitive Data Redaction**: Automatic redaction of API keys and personal data
- **Request Tracing**: Full request lifecycle tracking for debugging

# API Endpoints (Comprehensive Specification)

All API endpoints use RESTful conventions with JSON request/response bodies. Authentication is handled via JWT tokens in httpOnly cookies. Rate limiting is applied per user and endpoint.

## Standard Response Format

### Success Response
```json
{
  "success": true,
  "data": { /* endpoint-specific data */ },
  "pagination": { /* for list endpoints */ },
  "timestamp": "2024-01-15T10:30:00Z",
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
  "timestamp": "2024-01-15T10:30:00Z",
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

## Authentication Endpoints

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
      "createdAt": "2024-01-01T00:00:00Z"
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
    "buildTime": "2024-01-15T08:00:00Z",
    "commit": "abc123def456",
    "environment": "production"
  }
}
```

## Profile Management Endpoints

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
    "systemPrompt": "You are an expert coding assistant..."
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
        "systemPrompt": "You are an expert coding assistant..."
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

## Model Management Endpoints

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
    "jobId": "load_job_123",
    "status": "loading",
    "estimatedTime": 180
  }
}
```

**SSE Stream URL**: `/api/models/:modelId/load/stream?jobId=load_job_123`

## Thread Management Endpoints

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

### POST /api/threads

**Description**: Create a new thread
**Authentication**: Required
**Rate Limit**: 10 requests per minute per user

#### Request
```json
{
  "title": "New Discussion Topic"
}
```

#### Success Response (201)
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
      "updatedAt": "2024-01-15T10:50:00Z"
    }
  }
}
```

## Message Management Endpoints

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

### POST /api/threads/:threadId/models/:modelId/messages

**Description**: Send message to model and get response
**Authentication**: Required (thread owner only)
**Rate Limit**: 20 requests per minute per user

**URL Parameters:**
- `threadId`: Thread identifier
- `modelId`: Model identifier

#### Request
```json
{
  "content": "Explain the benefits of event-driven architecture",
  "temperature": 0.8,
  "stream": true
}
```

#### Non-streaming Response (200)
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

#### Streaming Response (SSE)
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
    "generatedAt": "2024-01-15T10:40:00Z",
    "modelUsed": "claude-opus"
  }
}
```

## Error Handling

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

## Rate Limiting Headers

All responses include rate limiting headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642234567
X-RateLimit-Window: 60
```