# Task 1.4: Model API Foundation

## Objective
Create model management API endpoints for an LLM chat application.

## Requirements
Build:
- Model catalog service with available models (Claude, ChatGPT, Google AI, Ollama)
- Model status tracking (available, loading, loaded, error)
- Model loading endpoints for local Ollama models
- Model configuration types and interfaces
- API endpoints: GET /api/models, GET /api/models/:id/status, POST /api/models/:id/load
- Error handling for model operations
- Timeout management (300s for model loading)

## Model Types
```typescript
interface Model {
  id: string;
  name: string;
  provider: 'claude' | 'openai' | 'google' | 'ollama';
  type: 'remote' | 'local';
  description: string;
  contextWindow: number;
  maxTokens: number;
  supportsStreaming: boolean;
  status: ModelStatus;
}

interface ModelStatus {
  state: 'available' | 'loading' | 'loaded' | 'error' | 'unavailable';
  progress?: number;
  error?: string;
  lastChecked: number;
  loadedAt?: number;
}
```

## API Endpoints
- GET /api/models - Return catalog of all available models
- GET /api/models/:id/status - Get current status of specific model
- POST /api/models/:id/load - Initiate loading of local model
- DELETE /api/models/:id/unload - Unload local model (optional)

## Model Catalog
- Claude: claude-3-sonnet, claude-3-haiku
- OpenAI: gpt-4, gpt-3.5-turbo
- Google AI: gemini-pro, gemini-pro-vision
- Ollama: gemma:2b (local deployment)

## Technical Requirements
- Model status persistence in memory/cache
- Ollama process management and health checking
- Timeout handling for long-running operations
- Progress tracking for model loading
- Resource usage monitoring
- Concurrent loading limitations

## Deliverables
- Model service with catalog management
- Status tracking and persistence
- API endpoints with proper error handling
- Ollama integration for local models
- Model loading progress tracking
- Timeout and resource management