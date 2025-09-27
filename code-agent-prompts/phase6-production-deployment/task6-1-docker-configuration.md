# Task 6.1: Docker Configuration

## Objective
Create production Docker setup for the application.

## Requirements
Configure:
- Multi-stage Dockerfile with Node.js and Ollama
- Supervisor configuration for process management
- Health check implementation
- Environment variable management
- Security hardening (non-root user, minimal base image)
- Build optimization for layer caching
- GitHub Container Registry push configuration

## Multi-Stage Dockerfile
```dockerfile
# Stage 1: Build Node.js application
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Extract Ollama binary
FROM ollama/ollama:latest AS ollama
# Extract ollama binary and dependencies

# Stage 3: Production runtime
FROM node:20-alpine AS runtime
# Combine Node.js app + Ollama with supervisor
```

## Process Management
- Supervisor configuration for managing multiple processes
- Node.js application process
- Ollama service process
- Log aggregation and rotation
- Graceful shutdown handling
- Process restart policies

## Security Hardening
- Non-root user creation and usage
- Minimal base image (Alpine Linux)
- Security scanning with Trivy
- Dependency vulnerability checking
- File system permissions optimization
- Network security configurations

## Environment Configuration
```dockerfile
ENV NODE_ENV=production
ENV PORT=8080
ENV OLLAMA_HOST=127.0.0.1:11434
ENV OLLAMA_MODELS=/app/models
ENV LOG_LEVEL=info
```

## Health Check Implementation
- HTTP health endpoint monitoring
- Ollama service availability check
- Database connectivity verification
- Resource utilization monitoring
- Graceful failure handling

## Build Optimization
- Layer caching for dependencies
- Multi-stage build efficiency
- .dockerignore for excluding unnecessary files
- Build argument optimization
- Image size minimization

## Supervisor Configuration
```ini
[supervisord]
nodaemon=true

[program:ollama]
command=ollama serve
autostart=true
autorestart=true

[program:node]
command=node dist/server.js
autostart=true
autorestart=true
```

## Container Features
- Model persistence volume mounting
- Log aggregation and streaming
- Resource limits (memory, CPU)
- Network port configuration
- Signal handling for graceful shutdown

## Development vs Production
- Development: Hot reload, debug tools
- Production: Optimized build, security hardening
- Multi-environment configuration
- Build-time vs runtime configuration

## Registry Configuration
- GitHub Container Registry (GHCR) setup
- Image tagging strategy (semantic versioning)
- Multi-architecture builds (AMD64, ARM64)
- Image signing and verification
- Registry authentication

## Monitoring Integration
- Prometheus metrics endpoint
- Log structured output
- Performance monitoring hooks
- Error tracking integration
- Resource usage reporting

## Backup and Recovery
- Model data backup strategies
- Configuration backup procedures
- Container state preservation
- Disaster recovery planning
- Database backup integration

## Deployment Strategy
- Blue-green deployment support
- Rolling update configuration
- Rollback procedures
- Health check validation
- Zero-downtime deployment

## Performance Optimization
- Memory allocation tuning
- CPU affinity configuration
- I/O optimization
- Caching strategies
- Resource limit enforcement

## Deliverables
- Multi-stage production Dockerfile
- Supervisor process management configuration
- Health check implementation
- Security hardening measures
- Build optimization for CI/CD
- GHCR integration and tagging strategy