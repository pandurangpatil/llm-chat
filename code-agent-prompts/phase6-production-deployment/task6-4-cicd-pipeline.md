# Task 6.4: CI/CD Pipeline

## Objective
Create complete CI/CD pipeline with GitHub Actions.

## Requirements
Build:
- Automated testing on pull requests
- Build and push Docker images to GHCR
- Staging deployment on stage branch
- Production deployment on main branch
- Post-deployment health checks
- Rollback on deployment failure
- Security scanning with Trivy
- Version tagging and release notes

## Pipeline Overview
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Validation  │→ │Integration  │→ │    Build    │→ │   Deploy    │
│ - Lint      │  │ - API Tests │  │ - Docker    │  │ - Cloud Run │
│ - Unit Test │  │ - E2E Tests │  │ - GHCR Push │  │ - Firebase  │
│ - Security  │  │ - Contract  │  │ - Version   │  │ - Health    │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
```

## Pull Request Workflow
```yaml
name: Pull Request Validation
on:
  pull_request:
    branches: [main, staging]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      # Install dependencies
      - run: npm ci

      # Linting and code quality
      - run: npm run lint
      - run: npm run type-check
      - run: npm run prettier:check

      # Unit tests with coverage
      - run: npm run test:unit
      - run: npm run test:coverage

      # Security scanning
      - run: npm audit --audit-level=moderate
      - run: npx trivy fs .
```

## Backend Build and Deploy
```yaml
name: Backend Deployment
on:
  push:
    branches: [main, staging]
    paths: ['backend/**', '.github/workflows/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:integration

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v3

      # Docker build and push
      - uses: docker/setup-buildx-action@v2
      - uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v4
        with:
          images: ghcr.io/${{ github.repository }}/api
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-

      - uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    if: github.ref == 'refs/heads/staging'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/deploy-cloudrun@v1
        with:
          service: llm-chat-api-staging
          image: ${{ needs.build.outputs.image-tag }}
          region: us-central1

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/deploy-cloudrun@v1
        with:
          service: llm-chat-api
          image: ${{ needs.build.outputs.image-tag }}
          region: us-central1
```

## Frontend Build and Deploy
```yaml
name: Frontend Deployment
on:
  push:
    branches: [main, staging]
    paths: ['frontend/**', '.github/workflows/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:components

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3

      # Build for environment
      - run: npm ci
      - run: npm run build:${{ github.ref_name }}

      # Deploy to Firebase
      - uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: '${{ secrets.GITHUB_TOKEN }}'
          firebaseServiceAccount: '${{ secrets.FIREBASE_SERVICE_ACCOUNT }}'
          channelId: live
          projectId: ${{ github.ref_name == 'main' && 'llm-chat-prod' || 'llm-chat-staging' }}
```

## End-to-End Testing
```yaml
name: E2E Testing
on:
  workflow_run:
    workflows: ["Backend Deployment", "Frontend Deployment"]
    types: [completed]
    branches: [staging]

jobs:
  e2e-tests:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci

      # Wait for deployment to be ready
      - run: sleep 60

      # Run E2E tests against staging
      - run: npm run test:e2e
        env:
          BASE_URL: https://staging.llmchat.app
          API_URL: https://api-staging.llmchat.app
```

## Security Scanning
```yaml
name: Security Scan
on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM
  push:
    branches: [main]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      # Scan filesystem
      - run: trivy fs .

      # Scan Docker image
      - run: trivy image ghcr.io/${{ github.repository }}/api:main

      # Upload results
      - uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

## Release Management
```yaml
name: Release
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      # Generate release notes
      - uses: release-drafter/release-drafter@v5
        with:
          config-name: release-drafter.yml
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # Build and tag Docker images
      - uses: docker/build-push-action@v4
        with:
          tags: |
            ghcr.io/${{ github.repository }}/api:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}/api:latest
```

## Health Check and Rollback
```yaml
name: Post-Deployment Health Check
on:
  workflow_run:
    workflows: ["Backend Deployment"]
    types: [completed]

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Wait for deployment
        run: sleep 120

      - name: Health check
        run: |
          for i in {1..5}; do
            if curl -f ${{ env.HEALTH_URL }}/health; then
              echo "Health check passed"
              exit 0
            fi
            sleep 30
          done
          echo "Health check failed"
          exit 1
        env:
          HEALTH_URL: ${{ github.ref_name == 'main' && 'https://api.llmchat.app' || 'https://api-staging.llmchat.app' }}

      - name: Rollback on failure
        if: failure()
        run: |
          # Implement rollback logic
          gcloud run services replace-traffic llm-chat-api \
            --to-revisions=PREVIOUS=100 \
            --region=us-central1
```

## Monitoring Integration
- Deployment success/failure notifications
- Performance metrics collection
- Error rate monitoring
- Cost tracking and alerts
- Security vulnerability notifications

## Environment Secrets
```bash
# GitHub Secrets Configuration
FIREBASE_SERVICE_ACCOUNT     # Firebase deployment credentials
GOOGLE_CLOUD_SA_KEY         # Google Cloud service account
DOCKER_REGISTRY_TOKEN       # GHCR authentication
SLACK_WEBHOOK_URL          # Notification webhook
SENTRY_AUTH_TOKEN          # Error tracking
```

## Notification Strategy
- Slack notifications for deployment status
- Email alerts for security issues
- GitHub commit status updates
- Dashboard integration for monitoring
- PagerDuty integration for critical failures

## Performance Monitoring
- Deployment time tracking
- Build performance metrics
- Test execution time monitoring
- Resource usage optimization
- Cost per deployment analysis

## Deliverables
- Complete GitHub Actions CI/CD pipeline
- Automated testing and validation
- Security scanning integration
- Deployment automation for staging/production
- Health check and rollback procedures
- Monitoring and notification setup