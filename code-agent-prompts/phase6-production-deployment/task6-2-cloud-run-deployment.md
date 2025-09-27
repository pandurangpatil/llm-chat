# Task 6.2: Google Cloud Run Deployment

## Objective
Set up Google Cloud Run deployment configuration.

## Requirements
Implement:
- Cloud Run service configuration (memory, CPU, scaling)
- Environment-specific configurations (staging/production)
- Secret Manager integration for sensitive data
- VPC connector setup
- Custom domain mapping
- Health check endpoints
- Blue-green deployment strategy
- Monitoring and logging configuration

## Cloud Run Service Configuration
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: llm-chat-api
  annotations:
    run.googleapis.com/ingress: all
    run.googleapis.com/execution-environment: gen2
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/memory: "4Gi"
        run.googleapis.com/cpu: "2"
        run.googleapis.com/max-scale: "2"
        run.googleapis.com/min-scale: "0"
    spec:
      containers:
      - image: ghcr.io/org/llm-chat:latest
        ports:
        - containerPort: 8080
        resources:
          limits:
            memory: "4Gi"
            cpu: "2000m"
```

## Environment Configurations
### Staging Environment
- Memory: 2Gi
- CPU: 1
- Min instances: 0
- Max instances: 1
- Concurrency: 50

### Production Environment
- Memory: 4Gi
- CPU: 2
- Min instances: 0
- Max instances: 2
- Concurrency: 100

## Secret Manager Integration
- JWT signing secrets
- Firebase admin service account
- AI provider API keys
- Database connection strings
- Encryption keys for user data

## VPC Configuration
- VPC connector for private resources
- Firewall rules for service communication
- Private IP allocation
- Network security policies
- Regional networking setup

## Custom Domain Setup
- SSL certificate provisioning
- Domain verification
- DNS configuration
- Load balancer configuration
- CDN integration (if applicable)

## Health Check Implementation
```typescript
// Health check endpoint
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    services: {
      database: await checkDatabaseHealth(),
      ollama: await checkOllamaHealth(),
      memory: process.memoryUsage(),
    }
  };
  res.json(health);
});
```

## Deployment Pipeline
1. Build and push Docker image to GHCR
2. Deploy to staging environment
3. Run health checks and integration tests
4. Deploy to production with traffic splitting
5. Monitor deployment and rollback if needed

## Blue-Green Deployment
- Traffic splitting configuration
- Gradual traffic migration
- Rollback procedures
- Health validation at each step
- Automated deployment triggers

## Monitoring and Logging
- Cloud Run metrics collection
- Application performance monitoring
- Error tracking and alerting
- Log aggregation and analysis
- Cost monitoring and optimization

## Auto-scaling Configuration
- Request-based scaling
- CPU and memory thresholds
- Cold start optimization
- Concurrency limits
- Regional scaling policies

## Security Configuration
- IAM roles and permissions
- Service account configuration
- Network security policies
- Container security scanning
- Runtime security monitoring

## Environment Variables
```bash
# Production environment
NODE_ENV=production
PORT=8080
DATABASE_URL=${SECRET_DATABASE_URL}
JWT_SECRET=${SECRET_JWT_SECRET}
FIREBASE_ADMIN_KEY=${SECRET_FIREBASE_ADMIN}
CLAUDE_API_KEY=${SECRET_CLAUDE_API_KEY}
OPENAI_API_KEY=${SECRET_OPENAI_API_KEY}
GOOGLE_AI_API_KEY=${SECRET_GOOGLE_AI_API_KEY}
```

## Cost Optimization
- Request-based billing optimization
- Resource limit tuning
- Cold start minimization
- Efficient scaling policies
- Reserved capacity planning

## Disaster Recovery
- Multi-region deployment strategy
- Database backup and restore
- Configuration backup procedures
- Recovery time objectives (RTO)
- Recovery point objectives (RPO)

## Performance Tuning
- Memory allocation optimization
- CPU usage monitoring
- Request timeout configuration
- Connection pooling setup
- Caching strategies

## Compliance and Governance
- Data residency requirements
- Audit logging configuration
- Compliance monitoring
- Policy enforcement
- Access control management

## Deliverables
- Cloud Run service configurations for staging/production
- Secret Manager integration setup
- Custom domain and SSL configuration
- Monitoring and alerting setup
- Deployment pipeline automation
- Documentation for operations and troubleshooting