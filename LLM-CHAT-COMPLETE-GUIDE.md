# LLM Chat Application - Complete Technical Guide

## Executive Summary

The LLM Chat Application is a multi-model AI chat platform that provides a unified interface for interacting with multiple LLM providers (Claude, ChatGPT, Google AI, and local Gemma 2B). Built for organizations requiring data sovereignty and cost-optimized AI usage, it features persistent conversation management, automatic summarization, and real-time streaming responses.

**Key Features:**
- Single unified interface for multiple AI models
- Private/secure self-hosted deployment
- Cost optimization through local model support
- Per-user encrypted API key management
- Thread-based conversation organization with auto-summarization
- Real-time streaming responses via Server-Sent Events

---

## System Architecture

### High-Level Design
```
┌─────────────────────┐    ┌─────────────────────┐
│   Frontend (React)  │◄──►│  Backend (Node.js)  │
│   Firebase Hosting  │    │   + Ollama Process  │
│                     │    │   Google Cloud Run  │
└─────────────────────┘    └─────────────────────┘
                                      │
                           ┌─────────────────────┐
                           │ Firebase Realtime   │
                           │     Database        │
                           └─────────────────────┘
```

### Component Overview
- **Frontend**: React 19 SPA with Vite, Material-UI 6, hosted on Firebase Hosting
- **Backend**: Node.js 22/Express server with Ollama integration on Google Cloud Run
- **Database**: Firebase Realtime Database with offline support
- **AI Integration**: Claude, OpenAI, Google AI APIs + local Ollama/Gemma 2B
- **Authentication**: JWT with httpOnly cookies
- **Real-time**: Server-Sent Events for message streaming

---

## Technical Stack

### Backend
- **Runtime**: Node.js 22 LTS with TypeScript 5
- **Framework**: Express.js with modular service architecture
- **Database**: Firebase Realtime Database (production) + Emulator (development)
- **AI Models**: Anthropic Claude, OpenAI GPT, Google AI, Ollama (Gemma 2B local)
- **Authentication**: JWT tokens with bcrypt password hashing
- **Deployment**: Docker containers on Google Cloud Run with auto-scaling

### Frontend
- **Framework**: React 19 with TypeScript 5 and Vite build system
- **UI Library**: Material-UI 6 with custom theme
- **State Management**: Zustand for client state management
- **Routing**: React Router v6 with protected routes
- **Styling**: Material-UI + Emotion for component styling
- **Deployment**: Static hosting on Firebase Hosting with CDN

### Infrastructure
- **Container Registry**: GitHub Container Registry (GHCR)
- **CI/CD**: GitHub Actions with automated testing and deployment
- **Secrets**: Google Cloud Secret Manager with KMS encryption
- **Monitoring**: Cloud Run metrics + Firebase Analytics
- **Domains**: Custom domains with HTTPS certificates

---

## Core Features & Implementation

### 1. Multi-Model AI Chat
- **Unified Interface**: Single chat interface supporting multiple AI providers
- **Model Switching**: Change models within the same thread (separate contexts)
- **Local Models**: Ollama integration for privacy-sensitive conversations
- **Streaming Responses**: Real-time token streaming via SSE

### 2. Thread-Based Conversations
- **Thread Management**: Organize conversations by topics/projects
- **Model Contexts**: Separate conversation history per model within threads
- **Auto-Titling**: 3-word titles generated from first message using local Ollama
- **Pagination**: Cursor-based pagination for efficient large dataset handling

### 3. Automatic Summarization
- **Trigger**: Auto-summarization after configurable message count (default: 10)
- **Processing**: Asynchronous, non-blocking summarization per conversation
- **Token Limit**: 300-700 token summaries maintained per model/thread
- **Manual Control**: Manual regeneration endpoint for user-initiated summaries

### 4. Authentication & Security
- **User Management**: Admin-only user creation (no self-registration)
- **JWT Authentication**: httpOnly cookies with secure flags
- **API Key Storage**: AES-256-GCM encryption with KMS-managed keys
- **Rate Limiting**: Per-user and per-endpoint request throttling
- **Input Validation**: Zod schemas for all API requests

### 5. Real-Time Communication
- **SSE Streaming**: Server-Sent Events for real-time message delivery
- **DB Watchers**: Firebase listeners with automatic cleanup
- **Timeout Management**: 30s message generation, 300s model loading
- **Resource Limits**: Max 100 concurrent watchers per instance

---

## API Design & Data Model

### Core API Endpoints
```
Authentication:
POST /api/login              - User authentication
POST /api/logout             - Session termination

Models:
GET  /api/models             - Available model catalog
GET  /api/models/:id/status  - Model loading status
POST /api/models/:id/load    - Load local model

Threads:
GET  /api/threads            - List user threads (paginated)
POST /api/threads            - Create new thread + first message
GET  /api/threads/:id        - Thread metadata
DELETE /api/threads/:id      - Delete thread

Messages:
GET  /api/threads/:id/models/:model/messages     - Get messages
POST /api/threads/:id/models/:model/messages     - Send message
GET  /api/threads/:id/models/:model/messages/:id - Stream response (SSE)

Profile:
GET  /api/profile            - User profile
PATCH /api/profile           - Update settings
POST /api/profile/api-keys   - Manage API keys

Health:
GET /api/health              - System health check
GET /api/version             - Version information
```

### Data Model Structure
```typescript
interface User {
  id: string;
  username: string;
  displayName: string;
  settings: {
    defaultModel: string;
    defaultTemperature: number;
    systemPrompt: string;
    summarizationInterval: number;
  };
  apiKeys: Record<string, EncryptedKey>;
}

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

---

## Development Workflow

### Test-Driven Development (TDD)
- **Red-Green-Refactor**: Write failing tests first, implement, then refactor
- **Coverage Requirements**: 80% minimum line coverage for critical paths
- **Testing Layers**: Unit (70%), Integration (25%), E2E (5%)
- **Contract-First**: API contracts defined before implementation

### CI/CD Pipeline
```
GitHub Actions Workflow:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Validation  │→ │Integration  │→ │    Build    │→ │   Deploy    │
│ - Lint      │  │ - API Tests │  │ - Docker    │  │ - Cloud Run │
│ - Unit Test │  │ - Newman    │  │ - GHCR Push │  │ - Firebase  │
│ - Security  │  │ - Contract  │  │ - Version   │  │ - Health    │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
```

### Environment Strategy
- **Development**: Local with Firebase Emulator Suite
- **Staging**: Integration testing environment (stage branch)
- **Production**: Live environment (main branch)

### Testing Tools
- **Backend**: Jest/Mocha, Supertest, Newman/Postman, Firebase Emulator
- **Frontend**: Vitest, React Testing Library, Playwright E2E
- **Quality Gates**: All tests pass, 80% coverage, no critical vulnerabilities

---

## Security & Performance

### Security Measures
- **Authentication**: JWT tokens with rotating secrets, httpOnly cookies
- **API Key Management**: AES-256-GCM encryption with Google KMS
- **Input Validation**: Comprehensive schema validation with Zod
- **Rate Limiting**: Per-user and per-IP request throttling
- **Network Security**: VPC integration, HTTPS enforcement
- **Dependency Security**: Automated vulnerability scanning with Trivy

### Performance Optimizations
- **Caching**: LRU caching, HTTP keep-alive, Docker layer caching
- **Pagination**: Cursor-based pagination for large datasets
- **Connection Pooling**: Efficient database and API connections
- **Streaming**: Real-time message delivery without polling
- **Resource Management**: Auto-scaling Cloud Run with scale-to-zero
- **Frontend**: Code splitting, virtualized scrolling, debounced search

### Monitoring & Health
- **Health Endpoints**: Real-time system status monitoring
- **Performance Metrics**: Response times, error rates, resource utilization
- **Deployment Validation**: Post-deployment health checks
- **Logging**: Structured JSON logging with correlation IDs

---

## Deployment & Infrastructure

### Container Strategy
```dockerfile
# Multi-stage build with Ollama integration
FROM node:20-alpine AS builder
# ... build Node.js application

FROM ollama/ollama:latest AS ollama
# ... extract Ollama binary

FROM node:20-alpine AS runtime
# ... combine Node.js + Ollama with supervisor
```

### Cloud Run Configuration
- **Staging**: 2Gi memory, 1 CPU, 0-2 instances
- **Production**: 4Gi memory, 2 CPU, 0-2 instances
- **Features**: Auto-scaling, health checks, blue-green deployment

### Firebase Hosting
- **CDN**: Global edge caching
- **SPA Routing**: Client-side routing support
- **Custom Domains**: HTTPS with automatic certificates
- **Environment Separation**: Staging and production sites

### Secret Management
- **Google Cloud Secret Manager**: JWT secrets, Firebase admin keys
- **GitHub Secrets**: OIDC authentication for CI/CD
- **Rotation Strategy**: Monthly JWT rotation, quarterly API key rotation

---

## Implementation Roadmap

### Phase 1: Foundation (Backend)
1. **Repository Setup**: Scaffolding, Docker, CI/CD skeleton
2. **Database Layer**: Firebase adapter, migrations, seeding
3. **Authentication**: User management, JWT implementation
4. **Model APIs**: Catalog, status, loading endpoints

### Phase 2: Core Services (Backend)
1. **Thread Management**: CRUD operations, pagination
2. **Message Processing**: Storage, context assembly, streaming
3. **AI Integration**: Provider abstraction, prompt building
4. **Summarization**: Async processing, job tracking

### Phase 3: Frontend Foundation
1. **Application Setup**: React, routing, Material-UI theme
2. **Authentication**: Login flow, session management
3. **Thread Interface**: Sidebar, creation, pagination
4. **Model Integration**: Selector, loading states

### Phase 4: Real-Time Features
1. **Streaming Chat**: SSE client, message display
2. **Model Loading**: Progress tracking, status indicators
3. **Health Monitoring**: System status, version display
4. **Profile Management**: Settings, API key configuration

### Phase 5: Integration & Testing
1. **Contract Testing**: API validation, mock servers
2. **Integration Tests**: End-to-end workflows
3. **Performance Testing**: Load testing, optimization
4. **Security Testing**: Vulnerability assessment

### Phase 6: Production Deployment
1. **Staging Deployment**: Environment setup, validation
2. **Production Release**: Coordinated deployment
3. **Monitoring Setup**: Health checks, alerting
4. **Documentation**: Operational procedures, troubleshooting

---

## Operational Procedures

### User Management
```bash
# Create user (development)
npm run add-user -- --username user@example.com --password <password>

# Production user creation via Cloud Function
gcloud functions call create-user --data '{"username":"user@example.com","password":"<password>"}'
```

### Model Management
- **Loading**: Models loaded on-demand per Cloud Run instance
- **Status Tracking**: Real-time loading progress via API
- **Resource Limits**: Memory and CPU optimized for Gemma 2B
- **Scaling**: Instances limited to 0-2 for cost control

### Health Monitoring
- **Endpoints**: `/api/health` for system status
- **Metrics**: Database connectivity, Ollama status, resource usage
- **Alerting**: Automated notifications for service degradation
- **Recovery**: Automatic rollback on health check failures

### Backup & Recovery
- **Database**: Firebase automatic backups
- **Configuration**: Infrastructure as Code with version control
- **Secrets**: Google Cloud Secret Manager with regional replication
- **Images**: Container images stored in GHCR with versioning

---

## Success Metrics

### Technical KPIs
- **Uptime**: 99.9% service availability
- **Response Time**: <500ms API response times
- **Test Coverage**: 80% minimum code coverage
- **Security**: Zero critical vulnerabilities in production

### User Experience KPIs
- **Message Latency**: <2s for first token, <10s for completion
- **Model Loading**: <5 minutes for local model initialization
- **Concurrent Users**: Support 100+ simultaneous conversations
- **Data Integrity**: Zero message loss or corruption

This comprehensive guide provides all essential information for implementing, deploying, and operating the LLM Chat Application. The architecture is designed for scalability, security, and maintainability while supporting the specific requirements of multi-model AI conversations with real-time streaming and automatic summarization capabilities.