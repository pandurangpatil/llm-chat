# Task 2.2: Message Processing Service

## Objective
Build message processing and storage system for an LLM chat application.

## Requirements
Implement:
- Message storage with thread and model association
- Message status tracking (pending, streaming, complete, failed)
- Context assembly for AI model conversations
- Token counting and management
- Message retrieval with pagination
- Message API endpoints: GET/POST /api/threads/:id/models/:model/messages
- Temperature and system prompt handling
- Message validation and sanitization

## Message Data Model
```typescript
interface Message {
  id: string;
  threadId: string;
  modelId: string;
  role: 'user' | 'assistant';
  content: string | string[]; // string array for streaming tokens
  tokens: number;
  temperature?: number;
  status: 'pending' | 'streaming' | 'complete' | 'failed';
  createdAt: number;
}
```

## API Endpoints
- GET /api/threads/:id/models/:model/messages - Retrieve conversation history
- POST /api/threads/:id/models/:model/messages - Send new message
- GET /api/threads/:id/models/:model/messages/:messageId - Stream AI response (SSE)

## Context Assembly
- Retrieve conversation history for specific model in thread
- Include summary if available (when message count > threshold)
- Apply system prompt from user settings
- Maintain token limits per model (context window management)
- Handle conversation trimming when approaching limits

## Message Processing Flow
1. Validate and sanitize user input
2. Store user message with 'pending' status
3. Assemble context for AI model
4. Initiate AI provider request
5. Stream response tokens and update message
6. Mark message as 'complete' when finished
7. Update thread metadata (message count, timestamp)

## Token Management
- Count tokens for both user and assistant messages
- Track cumulative token usage per thread/model
- Implement token limits and warnings
- Support for different tokenization methods per provider

## Validation Requirements
- Input sanitization for XSS prevention
- Message length limits (configurable)
- Rate limiting per user/thread
- Content filtering for inappropriate material
- Thread ownership verification

## Deliverables
- Message service with storage operations
- Context assembly logic
- Token counting and management
- API endpoints with streaming support
- Input validation and sanitization
- Error handling and recovery