# Task 2.4: Summarization Service

## Objective
Implement automatic conversation summarization service.

## Requirements
Create:
- Asynchronous summarization job processor
- Trigger system based on message count (configurable, default: 10)
- Summary generation using configured AI model
- Token limit enforcement (300-700 tokens)
- Summary storage per model/thread context
- Manual regeneration endpoint
- Background job tracking and status
- Non-blocking operation to avoid impacting chat performance

## Summarization Triggers
- Automatic: After configurable message count (default: 10)
- Manual: User-initiated summary regeneration
- Periodic: Scheduled summarization for active threads
- On-demand: API endpoint for immediate summarization

## Summary Generation Process
1. Detect summarization trigger (message count threshold)
2. Queue summarization job asynchronously
3. Retrieve conversation history for specific model/thread
4. Generate summary using designated AI model (prefer local Ollama)
5. Validate summary length (300-700 tokens)
6. Store summary with timestamp
7. Update thread metadata

## Summary Data Model
```typescript
interface Summary {
  threadId: string;
  modelId: string;
  content: string;
  tokens: number;
  messageCount: number;
  createdAt: number;
  generatedBy: string; // AI model used for generation
}
```

## Job Processing
- Background job queue with Redis/memory storage
- Job status tracking (pending, processing, completed, failed)
- Retry logic for failed summarizations
- Priority handling for user-requested summaries
- Resource limits to prevent system overload

## API Endpoints
- POST /api/threads/:id/models/:model/summarize - Manual summary generation
- GET /api/threads/:id/models/:model/summary - Retrieve current summary
- DELETE /api/threads/:id/models/:model/summary - Clear summary

## Configuration Options
- Message count threshold per user/global
- Summary length limits (min/max tokens)
- AI model preference for summarization
- Processing queue limits and timeouts
- Retry attempts and backoff strategies

## Performance Requirements
- Non-blocking summarization (async processing)
- Maximum 30 seconds per summary generation
- Support for 100+ concurrent summarization jobs
- Memory-efficient processing for large conversations
- Graceful degradation under high load

## Deliverables
- Summarization service with job processing
- Trigger system based on message counts
- Background job queue implementation
- Summary storage and retrieval
- Manual summarization endpoints
- Configuration management system