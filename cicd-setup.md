# CI/CD Setup Strategy - LLM Chat Platform

This document provides a comprehensive CI/CD implementation strategy specifically tailored for the LLM Chat platform's two-repository architecture, tech stack, and deployment requirements.

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [GitHub Actions Workflow Structure](#2-github-actions-workflow-structure)
3. [Backend CI/CD Pipeline Strategy](#3-backend-cicd-pipeline-strategy)
4. [Frontend CI/CD Pipeline Strategy](#4-frontend-cicd-pipeline-strategy)
5. [Container Strategy with Ollama](#5-container-strategy-with-ollama)
6. [Testing Integration Strategy](#6-testing-integration-strategy)
7. [Deployment Strategy](#7-deployment-strategy)
8. [Secret Management](#8-secret-management)
9. [Version Management & Release Strategy](#9-version-management--release-strategy)
10. [Environment Configuration](#10-environment-configuration)
11. [Monitoring & Health Checks](#11-monitoring--health-checks)
12. [Development Workflow Integration](#12-development-workflow-integration)
13. [Cross-Repository Coordination](#13-cross-repository-coordination)
14. [Performance & Cost Optimization](#14-performance--cost-optimization)
15. [Security Hardening](#15-security-hardening)

---

## 1. Architecture Overview

### Project-Specific Architecture

The LLM Chat platform uses a **dual-repository architecture** with distinct CI/CD pipelines:

```
┌─────────────────────┐    ┌─────────────────────┐
│   Backend Repo      │    │   Frontend Repo     │
│   (Node.js/Express) │    │   (React/Vite)      │
│                     │    │                     │
│ GitHub Actions      │    │ GitHub Actions      │
│        ↓            │    │        ↓            │
│ Docker Build        │    │ Vite Build          │
│        ↓            │    │        ↓            │
│ GHCR Push           │    │ Firebase Hosting    │
│        ↓            │    │                     │
│ Cloud Run Deploy    │    │                     │
└─────────────────────┘    └─────────────────────┘
```

### Key Architectural Decisions

- **Separate Repositories**: Independent versioning and deployment cycles
- **Container Strategy**: Single container with Node.js backend + Ollama process
- **Database Strategy**: Firebase Realtime Database (production) + Emulator Suite (CI/testing)
- **Registry Strategy**: GitHub Container Registry (GHCR) for Docker images
- **Deployment Strategy**: Google Cloud Run (backend) + Firebase Hosting (frontend)
- **Testing Strategy**: Firebase Emulator + Newman/Postman + Jest/Vitest

---

## 2. GitHub Actions Workflow Structure

### Workflow Triggers

#### Backend Repository Triggers
- **Push to main branch**: Automatic deployment pipeline
- **Pull request events**: Validation and testing pipeline
- **Manual workflow dispatch**: With version bump options (patch, minor, major)
- **Workflow file location**: `.github/workflows/backend-ci-cd.yml`

#### Frontend Repository Triggers
- **Push to main branch**: Automatic deployment pipeline
- **Pull request events**: Validation and testing pipeline
- **Manual workflow dispatch**: With version bump options (patch, minor, major)
- **Workflow file location**: `.github/workflows/frontend-ci-cd.yml`

### Job Orchestration Strategy

#### Backend Pipeline Jobs
1. **validate** - Linting, type checking, unit tests
2. **integration** - Firebase Emulator + Newman tests
3. **build** - Docker image with Ollama bundled
4. **security** - Vulnerability scanning, secret detection
5. **deploy** - Cloud Run deployment
6. **release** - GitHub release creation

#### Frontend Pipeline Jobs
1. **validate** - Linting, type checking, unit tests
2. **integration** - Backend mock integration tests
3. **build** - Vite production build
4. **deploy** - Firebase Hosting deployment
5. **release** - GitHub release creation

---

## 3. Backend CI/CD Pipeline Strategy

### Pipeline Stages Implementation

#### Stage 1: Code Validation
- **Environment**: Ubuntu latest with Node.js 20
- **Dependency management**: npm ci with caching
- **Code quality**: Linting, type checking, unit tests
- **Coverage reporting**: Generate and upload to Codecov
- **Required scripts**: `lint`, `type-check`, `test:unit`, `test:coverage`

#### Stage 2: Integration Testing
- **Dependencies**: Requires validation stage completion
- **Firebase Emulator**: Database and auth emulators on ports 9000, 5001, 8080
- **Database setup**: Run test migrations and seed data
- **Backend startup**: Start in CI mode with health check validation
- **API testing**: Execute Newman/Postman integration tests
- **Required scripts**: `migrate:test`, `seed:test`, `start:ci`, `wait-for-health`, `test:integration`, `test:newman`

#### Stage 3: Docker Build with Ollama
- **Dependencies**: Requires validation and integration stages
- **Version management**: Automatic patch bump or manual version selection
- **Registry**: GitHub Container Registry (GHCR)
- **Image tagging**: Semantic versioning with branch/PR tags
- **Build arguments**: Version, build time, commit hash
- **Caching**: GitHub Actions cache for layer optimization
- **Dockerfile**: Uses `Dockerfile.production` with Ollama integration
- **Output**: Image tag and digest for deployment stage

#### Stage 4: Security Scanning
- **Dependencies**: Requires Docker build completion
- **Dependency scanning**: npm audit with high-level vulnerability detection
- **Container scanning**: Trivy security scan with SARIF output
- **Secret detection**: TruffleHog for exposed credentials
- **Blocking criteria**: High/critical vulnerabilities prevent deployment

#### Stage 5: Cloud Run Deployment
- **Dependencies**: Requires build and security stages
- **Conditions**: Only deploys from main branch to production environment
- **Authentication**: Workload Identity Federation with Google Cloud
- **Service**: Deploy to `llm-chat-backend` in `us-central1` region
- **Environment variables**: NODE_ENV, VERSION
- **Secrets**: JWT_SECRET, FIREBASE_ADMIN_KEY, SYSTEM_ENCRYPTION_KEY from Secret Manager
- **Health validation**: Post-deployment health and version endpoint checks

### Backend-Specific Considerations

#### Ollama Integration in Container
- **Model Packaging**: Bundle Gemma 2B model files in Docker image or fetch at runtime
- **Process Management**: Supervisor script to manage Node.js + Ollama processes
- **Resource Allocation**: Configure Cloud Run CPU/memory for model loading
- **Startup Optimization**: Model pre-loading strategies vs on-demand loading

#### Firebase Emulator Integration
- **Database Testing**: Use emulator for deterministic integration tests
- **Seed Data**: Consistent test data for Newman/Postman collections
- **Admin SDK**: Service account configuration for emulator mode
- **Real-time Features**: Test SSE streaming with emulator

---

## 4. Frontend CI/CD Pipeline Strategy

### Pipeline Stages Implementation

#### Stage 1: Code Validation
- **Environment**: Ubuntu latest with Node.js 20 and npm caching
- **Quality checks**: Linting, type checking, unit tests, coverage reports
- **Coverage upload**: Codecov integration
- **Required scripts**: `lint`, `type-check`, `test:unit`, `test:coverage`

#### Stage 2: Integration Testing
- **Dependencies**: Requires validation stage completion
- **Mock testing**: Start mock backend server for isolated integration tests
- **Real backend testing**: Test against deployed staging backend
- **E2E testing**: Playwright browser automation tests
- **Required scripts**: `mock-server:start`, `test:integration:mock`, `test:integration:real`, `test:e2e`

#### Stage 3: Build Production
- **Dependencies**: Requires validation and integration stages
- **Version management**: Automatic patch bump or manual version selection
- **Environment variables**: Inject VITE_APP_VERSION and VITE_BACKEND_URL
- **Build process**: Vite production build with optimizations
- **Artifacts**: Upload dist folder for deployment stage
- **Output**: Version number for subsequent stages

#### Stage 4: Firebase Hosting Deployment
- **Dependencies**: Requires build stage completion
- **Conditions**: Only deploys from main branch to production environment
- **Artifact handling**: Download dist folder from build stage
- **Firebase integration**: Deploy to live channel with service account authentication
- **Required secrets**: GITHUB_TOKEN, FIREBASE_SERVICE_ACCOUNT, FIREBASE_PROJECT_ID
- **Post-deployment**: Health check validation after 30-second delay

### Frontend-Specific Considerations

#### Vite Build Optimization
- **Environment Variables**: Inject backend URLs, version info, feature flags
- **Bundle Analysis**: Track bundle size changes, optimize imports
- **Asset Optimization**: Image compression, font subsetting
- **Cache Management**: Configure caching headers for Firebase Hosting

#### Cross-Repository Integration
- **Backend Coordination**: Use staging backend for integration tests
- **API Contract Validation**: Ensure frontend changes align with backend contracts
- **Feature Flags**: Coordinate feature releases between repositories

---

## 5. Container Strategy with Ollama

### Dockerfile Strategy

#### Multi-Stage Build Approach
- **Stage 1**: Node.js application build with production dependencies
- **Stage 2**: Ollama binary extraction from official image
- **Stage 3**: Final production container combining Node.js + Ollama
- **Base images**: node:20-alpine for size optimization
- **Process management**: Supervisor for Node.js and Ollama coordination
- **Model handling**: Option to bundle Gemma 2B or download at runtime
- **Health checks**: 30-second intervals with 2-minute startup period
- **Port exposure**: 3000 for Node.js application

#### Supervisor Configuration
- **Daemon mode**: Non-daemon mode for container compatibility
- **Backend process**: Node.js server with automatic restart
- **Ollama process**: Ollama serve command with automatic restart
- **Logging**: Separate log files for backend and Ollama processes
- **Process management**: Both processes start automatically and restart on failure

### Build Optimization Strategies

#### Layer Caching
- **Dependency Layers**: Separate package.json changes from source code
- **Model Caching**: Cache downloaded models across builds
- **Multi-platform Builds**: Support AMD64 and ARM64 architectures

#### Size Optimization
- **Alpine Base**: Use minimal Alpine Linux images
- **Model Management**: Download models at runtime vs build time
- **Cleanup**: Remove build tools and temporary files

---

## 6. Testing Integration Strategy

### Test-Driven Development Integration

#### Unit Testing Pipeline
- **Matrix strategy**: Test against Node.js versions 18 and 20
- **Backend testing**: Jest with coverage reports and CI-optimized settings
- **Frontend testing**: Vitest with JUnit reporting and coverage
- **Configuration**: No watch mode for CI, coverage reporting enabled

#### Integration Testing with Firebase Emulator
- **Emulator setup**: Start Firebase database and auth emulators
- **Port configuration**: Database on 9000, Auth on 9099
- **Database preparation**: Run test migrations and seed deterministic data
- **Backend configuration**: Environment variables for emulator connectivity
- **API testing**: Newman execution with Postman collections and JUnit reporting
- **Required scripts**: `db:migrate:test`, `db:seed:test`, `start:test`
- **Test artifacts**: API test collection and CI environment configuration

#### Contract Testing Strategy
- **Spec generation**: Automatic OpenAPI specification from code
- **Response validation**: Test API responses against contract definitions
- **Breaking change detection**: Automated analysis of contract modifications
- **Required scripts**: `generate:openapi-spec`, `test:contract`, `contract:breaking-changes`

### Test Data Management

#### Database Seeding Strategy
- **Deterministic Data**: Consistent test data across environments
- **User Fixtures**: Test users with various API key configurations
- **Thread Scenarios**: Sample conversations for different models
- **Model Configuration**: Complete models_config seed data

#### Test Isolation
- **Database Cleanup**: Reset state between test runs
- **API Key Isolation**: Separate test API keys for each test suite
- **Model State**: Reset Ollama model loading state

---

## 7. Deployment Strategy

### Google Cloud Run Configuration

#### Service Configuration
- **Service name**: llm-chat-backend
- **Ingress**: Allow all traffic
- **CPU configuration**: 2 CPU limit, 1 CPU request, no throttling
- **Memory configuration**: 4Gi limit, 2Gi request (optimized for Ollama)
- **Scaling**: 0-10 instances with scale-to-zero capability
- **Container port**: 3000 for Node.js application
- **Environment**: Production NODE_ENV with PORT configuration
- **Image source**: GHCR with latest tag from CI/CD pipeline

#### Deployment Strategy
- **Blue-Green Deployment**: Zero-downtime deployments with traffic switching
- **Health Checks**: Comprehensive health endpoints for deployment validation
- **Rollback Strategy**: Automatic rollback on health check failures
- **Scaling Configuration**: Optimize for Ollama model loading requirements

### Firebase Hosting Configuration

#### Hosting Configuration
- **Public directory**: dist folder from Vite build
- **Ignored files**: firebase.json, hidden files, node_modules
- **SPA routing**: All routes redirect to index.html for client-side routing
- **Caching headers**: Long-term caching (1 year) for JS/CSS assets
- **File location**: firebase.json in project root

#### Deployment Features
- **Preview Channels**: PR-based preview deployments
- **CDN Distribution**: Global edge caching
- **Custom Domain**: HTTPS with automatic certificate management
- **Analytics Integration**: Firebase Analytics and Performance Monitoring

---

## 8. Secret Management

### GitHub Secrets Configuration

#### Backend Repository Secrets
**Authentication:**
- GITHUB_TOKEN: GHCR access token
- GCP_PROJECT_ID: Google Cloud Project identifier
- WIF_PROVIDER: Workload Identity Federation provider
- WIF_SERVICE_ACCOUNT: Service Account for deployment

**Application Secrets:**
- JWT_SECRET: JWT signing key
- FIREBASE_ADMIN_KEY: Firebase Admin SDK credentials
- SYSTEM_ENCRYPTION_KEY: API key encryption key

**External Services (Optional for Testing):**
- CLAUDE_API_KEY, OPENAI_API_KEY, GOOGLE_AI_API_KEY

#### Frontend Repository Secrets
**Firebase:**
- FIREBASE_TOKEN: Firebase CLI token
- FIREBASE_PROJECT_ID: Firebase project identifier
- FIREBASE_SERVICE_ACCOUNT: Service account for hosting

**Backend Integration:**
- BACKEND_URL_PRODUCTION: Production backend URL
- BACKEND_URL_STAGING: Staging backend URL

### Google Cloud Secret Manager Integration

#### Secret Configuration
**Google Cloud Secret Manager Setup:**
- **jwt-secret**: JWT signing key with automatic replication
- **firebase-admin**: Firebase Admin SDK credentials with automatic replication
- **system-key**: System encryption key with automatic replication
- **Configuration**: Terraform-managed secrets with automatic regional replication

### Secret Rotation Strategy

#### Automated Rotation
- **JWT Secrets**: Monthly rotation with graceful transition
- **API Keys**: Quarterly rotation with user notification
- **System Keys**: Manual rotation with coordinated deployment

---

## 9. Version Management & Release Strategy

### Semantic Versioning Implementation

#### Automated Version Bumping
- **Script location**: scripts/version-bump.js
- **Version detection**: Read current version from package.json
- **Bump types**: patch (default), minor, major based on environment variable
- **Semantic versioning**: Uses semver package for version increment
- **Package update**: Automatically updates package.json with new version
- **Logging**: Console output for version change tracking

#### Release Notes Generation
- **Tool**: release-drafter GitHub Action
- **Configuration**: .github/release-drafter.yml
- **Version injection**: Automatic version from CI pipeline
- **Authentication**: GITHUB_TOKEN for repository access
- **Content**: Auto-generated from PR labels and commit messages

### Coordinated Releases

#### Cross-Repository Versioning
- **Independent Versioning**: Each repository maintains its own version
- **Compatibility Matrix**: Document frontend/backend version compatibility
- **Release Coordination**: Use GitHub releases to coordinate deployments

#### Hotfix Strategy
- **Emergency Patches**: Direct-to-main with immediate deployment
- **Rollback Procedures**: Quick revert to previous stable version
- **Communication**: Automated notifications for emergency releases

---

## 10. Environment Configuration

### Multi-Environment Setup

#### Environment Hierarchy
- **Development**: Local development with Firebase Emulator
- **Staging**: Integration testing environment
- **Production**: Live production environment
- **Flow**: Development → Staging → Production with validation gates

#### Environment-Specific Configuration

##### Development Environment
- **NODE_ENV**: development
- **Firebase**: Local emulator URLs (database: 9000, auth: 9099)
- **Ollama**: Local host on port 11434
- **Logging**: Debug level for detailed output

##### Staging Environment
- **NODE_ENV**: staging
- **Firebase**: llm-chat-staging project
- **Logging**: Info level
- **CORS**: staging.llm-chat.web.app origin

##### Production Environment
- **NODE_ENV**: production
- **Firebase**: llm-chat-prod project
- **Logging**: Warn level for minimal output
- **CORS**: llm-chat.web.app origin

### Configuration Management

#### Environment Variable Strategy
- **Build-Time Variables**: Version info, feature flags
- **Runtime Variables**: API endpoints, database connections
- **Secret Variables**: API keys, certificates managed by Secret Manager

#### Feature Flag Implementation
- **File location**: utils/feature-flags.js
- **ENABLE_LOCAL_MODELS**: Boolean flag for Ollama/Gemma support
- **ENABLE_STREAMING**: SSE streaming functionality (default: true)
- **ENABLE_SUMMARIZATION**: Automatic conversation summarization
- **Configuration**: Environment variable based with sensible defaults

---

## 11. Monitoring & Health Checks

### Application Health Monitoring

#### Health Check Endpoints
- **Endpoint**: GET /api/health
- **Response data**: status, timestamp, version, uptime
- **Service checks**: database connectivity, Ollama status, memory usage, CPU usage
- **Status codes**: 200 for healthy, 503 for unhealthy
- **Monitoring**: Individual service status aggregation
- **Implementation**: Async checks with timeout handling

#### Deployment Validation
- **Stability wait**: 60-second delay for service startup
- **Health check loop**: 10 attempts with 10-second intervals
- **Endpoint validation**: curl requests to /api/health
- **Version verification**: Compare deployed version with expected version
- **Failure handling**: Exit with error code on validation failure
- **Monitoring**: Real-time deployment validation feedback

### Monitoring Integration

#### Cloud Run Monitoring
- **Request Metrics**: Latency, error rate, request count
- **Resource Metrics**: CPU, memory, instance count
- **Custom Metrics**: Model loading time, SSE connection count

#### Firebase Monitoring
- **Hosting Metrics**: Page load times, error rates
- **Performance Monitoring**: Core Web Vitals
- **Analytics**: User engagement, feature usage

---

## 12. Development Workflow Integration

### Branch Protection Rules

#### Main Branch Protection
- **Required status checks**: validate, integration, security (strict mode)
- **Pull request reviews**: 1 required approval with stale review dismissal
- **Code owner reviews**: Required for protected paths
- **Admin enforcement**: Disabled for emergency fixes
- **Push restrictions**: Limited to core-team members
- **Configuration file**: .github/branch-protection.yml

### Pull Request Automation

#### PR Workflow
- **Validation checks**: Linting, unit tests, fast integration tests
- **Security scanning**: npm audit with high-level vulnerability detection
- **Build verification**: Ensure code compiles successfully
- **Automated comments**: PR status updates via GitHub API
- **Required scripts**: `lint`, `test:unit`, `test:integration:fast`, `build`
- **Workflow file**: .github/workflows/pr.yml

### Code Quality Gates

#### Quality Metrics
- **Test Coverage**: Minimum 80% line coverage
- **Code Complexity**: Maximum cyclomatic complexity of 10
- **Dependency Freshness**: Alert on outdated dependencies
- **Security Vulnerabilities**: Block on high/critical issues

---

## 13. Cross-Repository Coordination

### Integration Testing Strategy

#### Backend API Contract Validation
- **Spec generation**: OpenAPI specification from backend code
- **Artifact upload**: Store spec for frontend repository access
- **Cross-repo notification**: Repository dispatch event to frontend
- **Payload data**: Version and spec artifact URL
- **Required token**: PAT_TOKEN for cross-repository communication
- **Required scripts**: `generate:openapi`

#### Frontend Contract Validation
- **Trigger**: Repository dispatch from backend repository
- **Spec download**: Retrieve OpenAPI spec from backend artifact
- **Contract testing**: Validate frontend implementation against spec
- **Result reporting**: POST validation results back to backend via webhook
- **Required scripts**: `test:contract` with spec parameter
- **Required secrets**: WEBHOOK_URL for result reporting

### Deployment Coordination

#### Coordinated Release Process
1. **Backend Release**: Deploy new API version to staging
2. **Frontend Validation**: Test frontend against staging backend
3. **Production Deployment**: Deploy backend to production
4. **Frontend Deployment**: Deploy frontend to production
5. **Post-Deployment Validation**: Run integration tests

---

## 14. Performance & Cost Optimization

### Build Performance

#### Caching Strategies
- **Docker layer caching**: GitHub Actions cache for build optimization
- **npm package caching**: Cache node_modules based on package-lock.json hash
- **Cache keys**: OS-specific with fallback restore keys
- **Cache mode**: Maximum mode for Docker layer caching
- **Performance impact**: Significant build time reduction

#### Parallel Execution
- **Matrix builds**: Multiple Node.js versions (18, 20) and test suites
- **Concurrent jobs**: Independent execution of unit, integration, and security tests
- **Resource optimization**: Parallel job execution reduces total pipeline time
- **Test suite separation**: unit, integration, e2e test categories
- **Runner allocation**: ubuntu-latest for consistent environments

### Cloud Run Cost Optimization

#### Resource Configuration
- **Memory limits**: 4Gi for Gemma 2B model requirements
- **Memory requests**: 1Gi minimum for startup
- **CPU limits**: 2 cores for model inference
- **CPU requests**: 0.5 cores for normal operation
- **Optimization**: Balanced for Ollama performance and cost

#### Scaling Configuration
- **Minimum scale**: 0 instances (scale-to-zero for cost optimization)
- **Maximum scale**: 5 instances (limit concurrent instances)
- **CPU throttling**: Disabled to maintain inference performance
- **Cost balance**: Optimize between availability and resource costs

---

## 15. Security Hardening

### Container Security

#### Vulnerability Scanning
- **Dependency scanning**: npm audit with high-level vulnerability detection
- **Container scanning**: Trivy security scanner with SARIF output format
- **Secret detection**: TruffleHog for exposed credentials and keys
- **Scanning scope**: Full repository with differential analysis
- **Output format**: SARIF for security tooling integration
- **Blocking criteria**: High/critical vulnerabilities prevent deployment

#### OIDC Authentication
- **Workload Identity Federation**: Eliminate long-lived service account keys
- **Provider configuration**: GitHub Actions provider in workload identity pool
- **Service account**: github-actions@project.iam.gserviceaccount.com
- **Security benefit**: Short-lived tokens instead of static credentials
- **Action**: google-github-actions/auth@v1

### Network Security

#### Cloud Run Security
- **Ingress control**: All traffic allowed (can be restricted to internal)
- **VPC integration**: Private VPC connector for secure networking
- **Egress control**: Private ranges only for enhanced security
- **Network isolation**: Secure communication with internal services

#### Firebase Security Rules
- **Rules version**: Version 2 for enhanced security features
- **User data access**: Read/write only for authenticated user's own data
- **Thread access**: Read/write only for thread owner (userId match)
- **Authentication requirement**: All operations require valid authentication
- **Data isolation**: Strict user-based data separation

### Supply Chain Security

#### Dependency Management
- **Dependabot configuration**: Automated dependency updates
- **Update schedule**: Weekly security and dependency updates
- **Package ecosystem**: npm for Node.js dependencies
- **Pull request limit**: Maximum 5 concurrent dependency PRs
- **Review assignment**: Security team for dependency reviews
- **Configuration file**: .github/dependabot.yml

#### Image Signing
- **Cosign integration**: Sigstore cosign for container image signing
- **Keyless signing**: Experimental mode with OIDC identity
- **Signature verification**: Cryptographic proof of image integrity
- **Supply chain security**: Enhanced container security posture
- **Action**: sigstore/cosign-installer@v3

---

## Implementation Roadmap

### Phase 1: Foundation
- Set up basic GitHub Actions workflows
- Configure GHCR and Cloud Run deployment
- Implement Firebase Emulator integration
- Set up basic health checks

### Phase 2: Testing Integration
- Integrate Newman/Postman API testing
- Set up contract testing framework
- Implement comprehensive health monitoring
- Configure security scanning

### Phase 3: Optimization
- Implement caching strategies
- Optimize build performance
- Set up monitoring and alerting
- Configure automated rollback

### Phase 4: Cross-Repository Coordination
- Implement contract validation between repos
- Set up coordinated deployment process
- Configure integration testing pipeline
- Implement feature flag coordination

This comprehensive CI/CD strategy provides a robust foundation for the LLM Chat platform, ensuring reliable deployments, comprehensive testing, and efficient development workflows while maintaining the specific requirements of the Node.js/Express + Ollama backend and React/Vite frontend architecture.