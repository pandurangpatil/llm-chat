# Frontend-Backend Integration Guide

This document outlines the integration patterns between the frontend and backend systems, including sequence diagrams for key user flows and technical implementation details.

## 1. Core Integration Patterns

The system uses an **async streaming architecture** where:
- API calls return immediately with tracking IDs (except title generation blocks briefly)
- LLM processing happens asynchronously in the background
- Real-time updates are delivered via Server-Sent Events (SSE)
- DB watchers handle token streaming with explicit deregistration on completion or disconnect

## 2. Authentication Flow

```mermaid
sequenceDiagram
    participant UI as Frontend
    participant API as Backend API
    participant DB as Database

    UI->>+API: POST /api/login
    Note over UI,API: {"username": "user@example.com", "password": "***"}
    API->>+DB: Validate credentials
    DB-->>-API: User data
    API->>API: Generate JWT token
    API-->>-UI: Set httpOnly cookie + user data
    Note over UI,API: Response: {"success": true, "data": {"user": {...}}}

    UI->>UI: Store user data in state
    UI->>UI: Redirect to dashboard
```

## 3. New Thread Creation with First Message

```mermaid
sequenceDiagram
    participant UI as Frontend
    participant API as Backend API
    participant DB as Database
    participant LLM as LLM Provider

    UI->>+API: POST /api/threads
    Note over UI,API: {<br/>  "title": "Architecture Discussion",<br/>  "firstMessage": {<br/>    "content": "Help me design a microservice",<br/>    "modelId": "claude-opus",<br/>    "temperature": 0.7<br/>  }<br/>}

    API->>+DB: Create thread
    DB-->>-API: threadId: "thread_123"

    API->>+DB: Store user message
    DB-->>-API: userMessageId: "msg_001"

    API->>+DB: Create assistant message placeholder
    DB-->>-API: assistantMessageId: "msg_002"

    par Async LLM Processing (fire-and-forget)
        API->>+LLM: Generate response
        LLM-->>API: Stream tokens
        API->>DB: Store tokens in message array
        LLM-->>-API: End of response
        API->>DB: Set message status='complete'
    end

    API->>+LLM: Call local Ollama for title (3 words)
    LLM-->>-API: Generated title: "Microservices Architecture Help"

    API-->>-UI: Return IDs and title
    Note over UI,API: {<br/>  "threadId": "thread_123",<br/>  "title": "Microservices Architecture Help",<br/>  "userMessageId": "msg_001",<br/>  "assistantMessageId": "msg_002",<br/>  "modelId": "claude-opus"<br/>}

    UI->>+API: GET /api/threads/thread_123/models/claude-opus/messages/msg_002 (SSE)
    Note over UI,API: Start SSE connection for streaming response

    API->>DB: Query existing tokens
    DB-->>API: Return stored tokens
    API-->>UI: Stream existing tokens
    API->>API: Register DB watcher

    loop New tokens arrive
        DB->>API: Notify new token (DB watcher)
        API-->>UI: SSE: token events
        Note over UI,API: event: token<br/>data: {"token": "Architecture", "index": N}
    end

    DB->>API: Message status='complete'
    API->>API: Deregister DB watcher
    API-->>UI: SSE: message_complete
    API-->>-UI: Close SSE connection

    UI->>UI: Display complete response
    UI->>UI: Update thread in sidebar
```

## 4. Continuing Conversation

```mermaid
sequenceDiagram
    participant UI as Frontend
    participant API as Backend API
    participant DB as Database
    participant LLM as LLM Provider

    UI->>+API: POST /api/threads/thread_123/models/claude-opus/messages
    Note over UI,API: {<br/>  "content": "What about data consistency?",<br/>  "temperature": 0.8<br/>}

    API->>+DB: Store user message
    DB-->>-API: userMessageId: "msg_003"

    API->>+DB: Create assistant message placeholder
    DB-->>-API: assistantMessageId: "msg_004"

    API-->>-UI: Return message IDs
    Note over UI,API: {<br/>  "userMessageId": "msg_003",<br/>  "assistantMessageId": "msg_004"<br/>}

    par Async Processing (fire-and-forget)
        API->>API: Build conversation context
        Note over API: 1. Fetch user system prompt<br/>2. Fetch thread summary for model<br/>3. Fetch recent messages<br/>4. Apply token budget

        API->>+LLM: Generate response with context
        LLM-->>API: Stream tokens

        loop For each token
            API->>DB: Append token to message array
        end

        LLM-->>-API: Complete
        API->>DB: Set message status='complete'

        API->>API: Trigger async summarization
        Note over API: Non-blocking summary update
    end

    UI->>+API: GET /api/threads/thread_123/models/claude-opus/messages/msg_004 (SSE)

    API->>DB: Query existing tokens
    DB-->>API: Return stored tokens
    API-->>UI: Stream existing tokens
    API->>API: Register DB watcher

    loop New tokens arrive
        DB->>API: Notify new token (DB watcher)
        API-->>UI: SSE: token event
    end

    DB->>API: Message status='complete'
    API->>API: Deregister DB watcher
    API-->>UI: SSE: message_complete
    API-->>-UI: Close SSE connection
```

## 5. Model Loading Flow

```mermaid
sequenceDiagram
    participant UI as Frontend
    participant API as Backend API
    participant Container as Container Runtime
    participant Ollama as Ollama Process

    UI->>+API: GET /api/models/gemma-2b-local/status
    API-->>-UI: {"status": "not_loaded"}

    UI->>UI: Show "Load Model" button

    UI->>+API: POST /api/models/gemma-2b-local/load
    API->>+Container: Start Ollama process
    Container->>+Ollama: Load gemma-2b model
    API-->>-UI: {"status": "loading"}

    UI->>UI: Show loading indicator

    loop Every 2 seconds
        UI->>+API: GET /api/models/gemma-2b-local/status
        Ollama-->>Container: Loading progress
        Container-->>API: Model status
        API-->>-UI: {"status": "loading", "progress": 65}
        UI->>UI: Update progress bar
    end

    Ollama-->>-Container: Model loaded
    Container-->>API: Model ready

    UI->>+API: GET /api/models/gemma-2b-local/status
    API-->>-UI: {"status": "loaded"}
    UI->>UI: Enable model for selection
```

## 6. Error Handling and Timeouts

```mermaid
sequenceDiagram
    participant UI as Frontend
    participant API as Backend API
    participant DB as Database
    participant Watcher as DB Watcher

    UI->>+API: GET /api/threads/thread_123/models/claude-opus/messages/msg_005 (SSE)

    API->>+DB: Query existing tokens
    DB-->>-API: Partial tokens
    API-->>UI: Stream existing tokens

    API->>+Watcher: Start watching message document
    Note over Watcher: Timeout: 30 seconds

    alt Normal completion
        Watcher-->>API: New token detected
        API-->>UI: SSE: token event
        Watcher-->>API: Message status='complete'
        API->>API: Deregister DB watcher
        API-->>UI: SSE: message_complete
        API-->>UI: Close connection

    else Client disconnects
        UI->>API: Close SSE connection
        API->>API: Deregister DB watcher
        API->>API: Clean up resources

    else Timeout scenario
        Watcher->>Watcher: 30 second timeout
        API->>API: Deregister DB watcher
        API-->>UI: SSE: error event
        Note over UI,API: {"error": "Generation timeout", "code": "TIMEOUT"}
        API-->>UI: Close connection with error
        UI->>UI: Show retry option

    else LLM API failure
        API->>DB: Mark message as failed
        API->>API: Deregister DB watcher
        API-->>UI: SSE: error event
        Note over UI,API: {"error": "LLM API failed", "code": "LLM_ERROR"}
        API-->>-UI: Close connection
        UI->>UI: Show error message
    end
```

## 7. Frontend State Management Integration

### Zustand Store Updates

```typescript
interface MessageState {
  // Streaming message tracking
  activeStreams: Map<string, {
    messageId: string;
    tokens: string[];
    isComplete: boolean;
    sseConnection?: EventSource;
  }>;

  // Actions
  startMessageStream: (messageId: string) => void;
  appendStreamingToken: (messageId: string, token: string) => void;
  completeMessage: (messageId: string, totalTokens: number) => void;
  handleStreamError: (messageId: string, error: Error) => void;
}
```

### Frontend SSE Client

```typescript
class MessageStreamClient {
  private eventSource: EventSource;
  private messageId: string;
  private timeout: number = 30000; // 30 seconds

  connect(threadId: string, modelId: string, messageId: string) {
    const url = `/api/threads/${threadId}/models/${modelId}/messages/${messageId}`;
    this.eventSource = new EventSource(url);

    this.eventSource.addEventListener('token', (event) => {
      const data = JSON.parse(event.data);
      this.onToken(data.token, data.index);
    });

    this.eventSource.addEventListener('message_complete', (event) => {
      const data = JSON.parse(event.data);
      this.onComplete(data.totalTokens);
      this.close();
    });

    this.eventSource.addEventListener('error', (event) => {
      const data = JSON.parse(event.data);
      this.onError(new Error(data.error));
      this.close();
    });

    // Client-side timeout
    setTimeout(() => {
      if (this.eventSource.readyState === EventSource.OPEN) {
        this.onTimeout();
        this.close();
      }
    }, this.timeout);
  }

  close() {
    this.eventSource?.close();
  }
}
```

## 8. Performance Considerations

### Connection Management
- Limit concurrent SSE connections per user (max 3)
- Implement connection pooling for backend
- Use HTTP/2 for multiplexed connections
- Explicit DB watcher deregistration to prevent resource leaks

### Memory Management
- Clean up completed message streams from frontend state
- Implement token array size limits in DB (max 10,000 tokens)
- Use cursor-based pagination for message history
- Deregister DB watchers on completion, disconnect, or timeout
- Track active watchers to prevent resource exhaustion

## 10. Testing Strategy

### Integration Tests
- Mock LLM providers for deterministic responses
- Test DB watcher timeout scenarios
- Verify SSE connection cleanup
- Test concurrent message streaming

### End-to-End Tests
- Full conversation flows with real streaming
- Model loading and switching scenarios
- Network failure recovery
- Performance under load

## 11. Security Considerations

### SSE Security
- JWT validation on SSE connections
- Rate limiting per user and IP
- Message ownership verification
- XSS protection in token content

### Data Privacy
- Token content sanitization
- Secure message storage encryption
- Audit logging for message access
- API key encryption in transit and at rest
