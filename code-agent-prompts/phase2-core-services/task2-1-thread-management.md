# Task 2.1: Thread Management Service

## Objective
Implement thread management system for a multi-model chat application.

## Requirements
Create:
- Thread CRUD operations (create, read, update, delete)
- Thread-user association and validation
- Cursor-based pagination for thread listing
- Thread metadata management (title, timestamps, model contexts)
- Auto-titling service using Ollama for 3-word titles from first message
- Thread API endpoints: GET/POST /api/threads, GET/DELETE /api/threads/:id
- Thread-model context separation
- Database queries optimized for thread operations

## Thread Data Model
```typescript
interface Thread {
  id: string;
  userId: string;
  title: string;
  createdAt: number;
  updatedAt: number;
  models: {
    [modelId: string]: {
      messageCount: number;
      lastMessageAt: number;
      summary?: string;
      summaryUpdatedAt?: number;
    };
  };
}
```

## API Endpoints
- GET /api/threads - List user threads with pagination
- POST /api/threads - Create new thread with optional first message
- GET /api/threads/:id - Get thread metadata
- DELETE /api/threads/:id - Delete thread and all associated messages
- PATCH /api/threads/:id - Update thread (title, settings)

## Pagination Requirements
- Cursor-based pagination using createdAt timestamp
- Default page size: 20 threads
- Support for next/previous cursors
- Total count estimation
- Sorting by most recent activity

## Auto-Titling Service
- Generate 3-word titles from first user message
- Use local Ollama model for title generation
- Fallback to "New Conversation" if generation fails
- Async processing to avoid blocking thread creation
- Title regeneration capability

## Business Rules
- Users can only access their own threads
- Thread deletion is soft delete with retention period
- Model contexts are isolated within threads
- Thread updates only allow title changes
- Maximum thread limit per user (configurable)

## Deliverables
- Thread service with CRUD operations
- Pagination implementation
- Auto-titling service integration
- API endpoints with validation
- Database optimization for queries
- Error handling and logging