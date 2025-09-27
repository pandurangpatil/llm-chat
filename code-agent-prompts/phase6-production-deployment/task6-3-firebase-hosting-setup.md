# Task 6.3: Firebase Hosting Setup

## Objective
Configure Firebase Hosting for React frontend.

## Requirements
Set up:
- Firebase hosting configuration
- Build and deployment scripts
- Environment-specific builds (staging/production)
- CDN and caching rules
- SPA routing configuration
- Custom domain setup with SSL
- GitHub Actions deployment workflow
- Rollback procedures

## Firebase Configuration
```json
{
  "hosting": [
    {
      "target": "production",
      "public": "dist",
      "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
      "rewrites": [
        {
          "source": "**",
          "destination": "/index.html"
        }
      ],
      "headers": [
        {
          "source": "/static/**",
          "headers": [
            {
              "key": "Cache-Control",
              "value": "public, max-age=31536000, immutable"
            }
          ]
        }
      ]
    },
    {
      "target": "staging",
      "public": "dist",
      "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
      "rewrites": [
        {
          "source": "**",
          "destination": "/index.html"
        }
      ]
    }
  ]
}
```

## Build Configuration
### Environment-Specific Builds
- Development: Hot reload, debug tools
- Staging: Production build with staging API endpoints
- Production: Optimized build with production API endpoints

### Build Scripts
```json
{
  "scripts": {
    "build:staging": "vite build --mode staging",
    "build:production": "vite build --mode production",
    "deploy:staging": "firebase deploy --only hosting:staging",
    "deploy:production": "firebase deploy --only hosting:production"
  }
}
```

## Caching Strategy
- Static assets: 1 year cache with immutable flag
- HTML files: No cache (always fresh)
- API responses: Short-term cache with revalidation
- Images and media: Long-term cache with versioning

## SPA Routing Configuration
- Catch-all routing for client-side navigation
- 404 handling for non-existent routes
- Deep linking support
- SEO optimization with meta tags
- Social media sharing configuration

## Custom Domain Setup
- Domain verification process
- SSL certificate provisioning
- DNS configuration (A/CNAME records)
- Subdomain routing (api.domain.com, app.domain.com)
- HTTPS enforcement and redirects

## Environment Configuration
```typescript
// vite.config.ts
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');

  return {
    define: {
      __API_URL__: JSON.stringify(env.VITE_API_URL),
      __ENVIRONMENT__: JSON.stringify(mode),
    },
    build: {
      outDir: 'dist',
      sourcemap: mode === 'development',
      minify: mode === 'production',
    },
  };
});
```

## Performance Optimization
- Code splitting and lazy loading
- Bundle size optimization
- Image optimization and compression
- Font loading optimization
- Critical CSS inlining
- Resource preloading

## Security Headers
```json
{
  "headers": [
    {
      "source": "**",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        },
        {
          "key": "Strict-Transport-Security",
          "value": "max-age=31536000; includeSubDomains"
        }
      ]
    }
  ]
}
```

## GitHub Actions Workflow
```yaml
name: Deploy Frontend
on:
  push:
    branches: [main, staging]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run build:${{ github.ref_name }}
      - uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: '${{ secrets.GITHUB_TOKEN }}'
          firebaseServiceAccount: '${{ secrets.FIREBASE_SERVICE_ACCOUNT }}'
          channelId: live
          projectId: llm-chat-app
```

## Monitoring and Analytics
- Firebase Analytics integration
- Performance monitoring
- Error tracking and reporting
- User behavior analytics
- Core Web Vitals monitoring

## CDN Configuration
- Global edge cache distribution
- Regional optimization
- Bandwidth optimization
- Compression algorithms (gzip, brotli)
- Cache invalidation strategies

## Staging Environment
- Separate Firebase project for staging
- Staging API endpoint configuration
- Test data and user accounts
- Feature flag integration
- A/B testing setup

## Rollback Procedures
- Version history tracking
- Quick rollback to previous versions
- Database migration rollback
- Configuration rollback
- Emergency deployment procedures

## SEO and Meta Tags
- Dynamic meta tag generation
- Open Graph tags for social sharing
- Twitter Card configuration
- Structured data markup
- Sitemap generation

## Accessibility Features
- WCAG compliance validation
- Screen reader optimization
- Keyboard navigation support
- High contrast mode
- Reduced motion preferences

## Environment Variables
```env
# Production
VITE_API_URL=https://api.llmchat.app
VITE_ENVIRONMENT=production
VITE_ANALYTICS_ID=G-XXXXXXXXXX

# Staging
VITE_API_URL=https://api-staging.llmchat.app
VITE_ENVIRONMENT=staging
VITE_ANALYTICS_ID=G-YYYYYYYYYY
```

## Deliverables
- Firebase hosting configuration for staging/production
- Build and deployment scripts
- Custom domain setup with SSL
- CDN and caching optimization
- GitHub Actions deployment workflow
- Performance monitoring and analytics setup