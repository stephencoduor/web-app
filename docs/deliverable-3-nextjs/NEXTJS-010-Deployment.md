# NEXTJS-010: Deployment & Infrastructure

## Status: Draft

## Last Updated: 2026-03-13

---

## 1. Docker Containerization

### 1.1 Next.js Standalone Output

Enable standalone output mode in Next.js to produce a self-contained deployment artifact that includes only the files needed to run in production.

```typescript
// next.config.ts
import createNextIntlPlugin from 'next-intl/plugin';

const withNextIntl = createNextIntlPlugin('./src/lib/i18n/request.ts');

const nextConfig = {
  output: 'standalone'
  // ... other config
};

export default withNextIntl(nextConfig);
```

### 1.2 Dockerfile

```dockerfile
# ---- Stage 1: Dependencies ----
FROM node:20-alpine AS deps
WORKDIR /app

# Install dependencies only when package files change
COPY package.json package-lock.json ./
RUN npm ci --production=false

# ---- Stage 2: Build ----
FROM node:20-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build-time environment variables
ARG NEXT_PUBLIC_FINERACT_API_URL
ARG NEXT_PUBLIC_TENANT_ID=default

ENV NEXT_PUBLIC_FINERACT_API_URL=$NEXT_PUBLIC_FINERACT_API_URL
ENV NEXT_PUBLIC_TENANT_ID=$NEXT_PUBLIC_TENANT_ID
ENV NEXT_TELEMETRY_DISABLED=1

RUN npm run build

# ---- Stage 3: Runner ----
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# Create non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy standalone output
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

# Set correct permissions
RUN chown -R nextjs:nodejs /app

USER nextjs

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

### 1.3 Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  nextjs-app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NEXT_PUBLIC_FINERACT_API_URL: http://fineract-backend:8443/fineract-provider
        NEXT_PUBLIC_TENANT_ID: default
    ports:
      - '3000:3000'
    environment:
      - FINERACT_API_URL=http://fineract-backend:8443/fineract-provider
      - AUTH_SECRET=your-auth-secret-at-least-32-chars
    depends_on:
      - fineract-backend
    networks:
      - msacco-network

  fineract-backend:
    image: apache/fineract:latest
    ports:
      - '8443:8443'
    environment:
      - FINERACT_DEFAULT_TENANTDB_HOSTNAME=mysql
      - FINERACT_DEFAULT_TENANTDB_PORT=3306
    depends_on:
      - mysql
    networks:
      - msacco-network

  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=mysql
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - msacco-network

  nginx:
    image: nginx:alpine
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - nextjs-app
      - fineract-backend
    networks:
      - msacco-network

networks:
  msacco-network:
    driver: bridge

volumes:
  mysql-data:
```

### 1.4 .dockerignore

```
node_modules
.next
.git
.github
*.md
e2e
coverage
.env*.local
```

### 1.5 Image Size Optimization

The multi-stage build produces a minimal image:

| Stage     | Purpose              | Included                                  |
| --------- | -------------------- | ----------------------------------------- |
| `deps`    | Install npm packages | `node_modules` only                       |
| `builder` | Build Next.js app    | Full source + build output                |
| `runner`  | Production runtime   | Standalone server + static files + public |

Expected image size: ~150-200 MB (alpine base + Node.js + app).

---

## 2. Environment Variables

### 2.1 Server-Side Variables (Runtime)

These variables are available only on the server. They are NOT exposed to the browser and can contain secrets.

| Variable             | Required | Default       | Description                                                                           |
| -------------------- | -------- | ------------- | ------------------------------------------------------------------------------------- |
| `FINERACT_API_URL`   | Yes      | --            | Full URL to Fineract backend (e.g., `https://fineract.example.com/fineract-provider`) |
| `AUTH_SECRET`        | Yes      | --            | Secret key for signing session tokens (min 32 chars)                                  |
| `AUTH_PROVIDER`      | No       | `basic`       | Authentication provider: `basic` or `oidc`                                            |
| `OIDC_ISSUER`        | If OIDC  | --            | OIDC provider issuer URL (e.g., Zitadel instance)                                     |
| `OIDC_CLIENT_ID`     | If OIDC  | --            | OIDC client ID                                                                        |
| `OIDC_CLIENT_SECRET` | If OIDC  | --            | OIDC client secret                                                                    |
| `OIDC_REDIRECT_URI`  | If OIDC  | --            | Callback URL (`https://app.example.com/callback`)                                     |
| `LOG_LEVEL`          | No       | `info`        | Logging level: `debug`, `info`, `warn`, `error`                                       |
| `SENTRY_DSN`         | No       | --            | Sentry error tracking DSN                                                             |
| `SENTRY_AUTH_TOKEN`  | No       | --            | Sentry auth token for source map uploads                                              |
| `REDIS_URL`          | No       | --            | Redis URL for session storage (production)                                            |
| `NODE_ENV`           | No       | `development` | `development`, `production`, `test`                                                   |

### 2.2 Client-Side Variables (Build-Time)

These are embedded at build time and available in browser code. They MUST be prefixed with `NEXT_PUBLIC_`.

| Variable                       | Required | Default   | Description                                    |
| ------------------------------ | -------- | --------- | ---------------------------------------------- |
| `NEXT_PUBLIC_FINERACT_API_URL` | Yes      | --        | Public Fineract API URL (may go through proxy) |
| `NEXT_PUBLIC_TENANT_ID`        | Yes      | `default` | Fineract tenant identifier                     |
| `NEXT_PUBLIC_APP_NAME`         | No       | `M-SACCO` | Application display name                       |
| `NEXT_PUBLIC_DEFAULT_LOCALE`   | No       | `en-US`   | Default locale                                 |
| `NEXT_PUBLIC_ENABLE_MSW`       | No       | `false`   | Enable Mock Service Worker for development     |
| `NEXT_PUBLIC_SENTRY_DSN`       | No       | --        | Sentry DSN for client-side error tracking      |

### 2.3 Environment Files

```bash
# .env.example -- Template for all deployments (committed to git)
# Server-side
FINERACT_API_URL=https://fineract.example.com/fineract-provider
AUTH_SECRET=change-me-to-a-secure-random-string-at-least-32-chars
AUTH_PROVIDER=basic
# OIDC_ISSUER=
# OIDC_CLIENT_ID=
# OIDC_CLIENT_SECRET=
# OIDC_REDIRECT_URI=
LOG_LEVEL=info
# SENTRY_DSN=
# REDIS_URL=

# Client-side (NEXT_PUBLIC_*)
NEXT_PUBLIC_FINERACT_API_URL=https://fineract.example.com/fineract-provider
NEXT_PUBLIC_TENANT_ID=default
NEXT_PUBLIC_APP_NAME=M-SACCO
NEXT_PUBLIC_DEFAULT_LOCALE=en-US
```

```bash
# .env.local -- Local development overrides (NOT committed)
FINERACT_API_URL=http://localhost:8443/fineract-provider
AUTH_SECRET=local-development-secret-at-least-32-chars
NEXT_PUBLIC_FINERACT_API_URL=http://localhost:8443/fineract-provider
NEXT_PUBLIC_TENANT_ID=default
NEXT_PUBLIC_ENABLE_MSW=false
```

```bash
# .env.production -- Production defaults (committed)
LOG_LEVEL=warn
NEXT_PUBLIC_ENABLE_MSW=false
```

### 2.4 Runtime Configuration for Server-Side Variables

For Docker deployments where `FINERACT_API_URL` needs to be set at runtime (not build time), use a server-side configuration module:

```typescript
// src/lib/config/server.ts
// Only imported in server-side code

export const serverConfig = {
  fineractApiUrl: process.env.FINERACT_API_URL!,
  authSecret: process.env.AUTH_SECRET!,
  authProvider: (process.env.AUTH_PROVIDER ?? 'basic') as 'basic' | 'oidc',
  oidc: {
    issuer: process.env.OIDC_ISSUER,
    clientId: process.env.OIDC_CLIENT_ID,
    clientSecret: process.env.OIDC_CLIENT_SECRET,
    redirectUri: process.env.OIDC_REDIRECT_URI
  },
  logLevel: process.env.LOG_LEVEL ?? 'info',
  sentryDsn: process.env.SENTRY_DSN
};

// Validate required variables at startup
if (!serverConfig.fineractApiUrl) {
  throw new Error('FINERACT_API_URL environment variable is required');
}
if (!serverConfig.authSecret || serverConfig.authSecret.length < 32) {
  throw new Error('AUTH_SECRET must be at least 32 characters');
}
```

---

## 3. Nginx Reverse Proxy Configuration

### 3.1 Basic Configuration

```nginx
# nginx/nginx.conf

upstream nextjs_upstream {
    server nextjs-app:3000;
}

upstream fineract_upstream {
    server fineract-backend:8443;
}

server {
    listen 80;
    server_name app.example.com;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # API proxy -- route Fineract API requests to backend
    location /fineract-provider/ {
        proxy_pass https://fineract_upstream/fineract-provider/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings for long-running Fineract operations
        proxy_connect_timeout 60s;
        proxy_send_timeout 120s;
        proxy_read_timeout 120s;

        # CORS headers (if needed for direct browser access)
        add_header Access-Control-Allow-Origin $http_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Authorization, Content-Type, Fineract-Platform-TenantId" always;

        if ($request_method = OPTIONS) {
            return 204;
        }
    }

    # Next.js static files (long cache)
    location /_next/static/ {
        proxy_pass http://nextjs_upstream;
        proxy_cache_valid 200 365d;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # Next.js image optimization
    location /_next/image {
        proxy_pass http://nextjs_upstream;
        proxy_cache_valid 200 60m;
    }

    # Public static assets
    location /assets/ {
        proxy_pass http://nextjs_upstream;
        proxy_cache_valid 200 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    # All other requests go to Next.js
    location / {
        proxy_pass http://nextjs_upstream;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### 3.2 Rate Limiting

```nginx
# Add to http block in nginx.conf
http {
    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

    server {
        # Apply rate limiting to API
        location /fineract-provider/ {
            limit_req zone=api burst=50 nodelay;
            # ... proxy config
        }

        # Stricter rate limiting for login
        location /api/auth/ {
            limit_req zone=login burst=3 nodelay;
            # ... proxy config
        }
    }
}
```

---

## 4. CI/CD Pipeline -- GitHub Actions

### 4.1 Complete Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test:run
      - run: npm run i18n:validate

  build:
    runs-on: ubuntu-latest
    needs: lint-and-test
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            NEXT_PUBLIC_FINERACT_API_URL=${{ vars.FINERACT_API_URL }}
            NEXT_PUBLIC_TENANT_ID=${{ vars.TENANT_ID }}

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          # Example: SSH deploy, kubectl apply, or cloud provider CLI
          echo "Deploying ${{ needs.build.outputs.image-tag }} to staging"
          # ssh deploy@staging "docker pull $IMAGE && docker-compose up -d"

  deploy-production:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to production"
          # ssh deploy@production "docker pull $IMAGE && docker-compose up -d"

  e2e-tests:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/develop'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - name: Run E2E tests against staging
        run: npx playwright test --project=chromium
        env:
          PLAYWRIGHT_BASE_URL: ${{ vars.STAGING_URL }}
          TEST_USERNAME: ${{ secrets.E2E_USERNAME }}
          TEST_PASSWORD: ${{ secrets.E2E_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: e2e-report
          path: playwright-report/
```

### 4.2 Branch Strategy

| Branch           | Environment | Trigger | E2E Tests            |
| ---------------- | ----------- | ------- | -------------------- |
| `develop`        | Staging     | Push    | After deploy         |
| `main`           | Production  | Push    | Pre-deploy (staging) |
| Feature branches | --          | PR only | Lint + unit tests    |

---

## 5. Vercel Deployment (Alternative)

For teams preferring a managed deployment platform, Vercel provides zero-configuration Next.js deployment.

### 5.1 Project Settings

```typescript
// next.config.ts -- Vercel-specific adjustments
const nextConfig = {
  // Do NOT set output: 'standalone' for Vercel
  // Vercel handles its own build optimization

  // Proxy Fineract API calls through Next.js rewrites
  async rewrites() {
    return [
      {
        source: '/fineract-provider/:path*',
        destination: `${process.env.FINERACT_API_URL}/:path*`
      }
    ];
  }
};
```

### 5.2 Environment Variables on Vercel

Set via Vercel Dashboard > Project > Settings > Environment Variables:

- `FINERACT_API_URL` -- Server-side Fineract URL
- `AUTH_SECRET` -- Auth signing secret
- `NEXT_PUBLIC_FINERACT_API_URL` -- Client-side API URL (use `/fineract-provider` with rewrites)
- `NEXT_PUBLIC_TENANT_ID` -- Tenant ID

### 5.3 Edge Middleware

On Vercel, `middleware.ts` runs at the Edge, which provides sub-millisecond auth checks globally:

```typescript
// middleware.ts -- same file works for both Docker and Vercel
export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|assets).*)']
};
```

### 5.4 Vercel vs Docker Comparison

| Aspect           | Vercel                   | Docker + Nginx         |
| ---------------- | ------------------------ | ---------------------- |
| Setup complexity | Low (automatic)          | Medium (manual config) |
| Cost             | Per-seat + usage pricing | Infrastructure cost    |
| Scaling          | Automatic                | Manual or K8s          |
| Edge middleware  | Global CDN edge          | Single region          |
| API proxy        | Via rewrites             | Via Nginx              |
| Custom domains   | Dashboard config         | DNS + cert management  |
| Data residency   | Vercel regions           | Full control           |
| Vendor lock-in   | Moderate                 | None                   |

---

## 6. Monitoring and Logging

### 6.1 Next.js Instrumentation

```typescript
// src/instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    // Server-side only instrumentation

    // Sentry server-side initialization
    if (process.env.SENTRY_DSN) {
      const Sentry = await import('@sentry/nextjs');
      Sentry.init({
        dsn: process.env.SENTRY_DSN,
        tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
        environment: process.env.NODE_ENV
      });
    }

    // Structured logging setup
    console.log(
      JSON.stringify({
        level: 'info',
        message: 'M-SACCO Next.js server starting',
        timestamp: new Date().toISOString(),
        version: process.env.npm_package_version
      })
    );
  }
}
```

### 6.2 Sentry Integration

```bash
npm install @sentry/nextjs
npx @sentry/wizard@latest -i nextjs
```

Configure in `sentry.client.config.ts` and `sentry.server.config.ts`:

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.05,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.replayIntegration(),
    Sentry.browserTracingIntegration()
  ]
});
```

```typescript
// sentry.server.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 0.1
});
```

### 6.3 Web Vitals Monitoring

```typescript
// app/layout.tsx (or a dedicated client component)
import { useReportWebVitals } from 'next/web-vitals';

function WebVitalsReporter() {
  useReportWebVitals((metric) => {
    // Send to analytics
    const body = JSON.stringify({
      name: metric.name, // CLS, FID, FCP, LCP, TTFB
      value: metric.value,
      rating: metric.rating, // "good", "needs-improvement", "poor"
      navigationType: metric.navigationType
    });

    // Send to your analytics endpoint
    if (navigator.sendBeacon) {
      navigator.sendBeacon('/api/vitals', body);
    }
  });

  return null;
}
```

### 6.4 Health Check Endpoint

```typescript
// app/api/health/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  const health = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version || 'unknown'
  };

  // Check Fineract connectivity
  try {
    const response = await fetch(`${process.env.FINERACT_API_URL}/api/v1/authentication`, { method: 'HEAD', signal: AbortSignal.timeout(5000) });
    health.fineract = response.ok ? 'connected' : 'error';
  } catch {
    health.fineract = 'unreachable';
  }

  const statusCode = health.fineract === 'connected' ? 200 : 503;
  return NextResponse.json(health, { status: statusCode });
}
```

---

## 7. Migration Cutover Strategy

### 7.1 Phase 1: Parallel Deployment (Week 1-2)

Run both Angular and Next.js apps behind the same Nginx:

```nginx
# Phase 1: Angular serves everything, Next.js on a separate path
location /next/ {
    proxy_pass http://nextjs-app:3000/;
}

location / {
    # Existing Angular app
    root /usr/share/nginx/html;
    try_files $uri $uri/ /index.html;
}

location /fineract-provider/ {
    proxy_pass https://fineract_upstream/fineract-provider/;
}
```

During this phase:

- Both apps run simultaneously
- Next.js is accessible at `/next/*` for testing
- No user-facing changes

### 7.2 Phase 2: Feature Flag Routing (Week 3-4)

Use cookies or user attributes to route specific users to Next.js:

```nginx
# Phase 2: Feature-flagged routing
map $cookie_nextjs_beta $backend {
    "true"  http://nextjs-app:3000;
    default http://angular-app:80;
}

# Or route specific modules
location /clients {
    proxy_pass http://nextjs-app:3000;
}

# Everything else stays on Angular
location / {
    root /usr/share/nginx/html;
    try_files $uri $uri/ /index.html;
}
```

During this phase:

- Internal users and beta testers use Next.js
- Remaining users stay on Angular
- Monitor error rates and performance

### 7.3 Phase 3: Full Cutover (Week 5)

```nginx
# Phase 3: Next.js serves all traffic
location / {
    proxy_pass http://nextjs-app:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}

# Keep legacy hash URLs redirected
location = / {
    if ($arg_hash) {
        return 301 $arg_hash;
    }
    proxy_pass http://nextjs-app:3000;
}
```

### 7.4 Rollback Plan

If issues are found during cutover:

1. **Immediate rollback (< 5 min):** Switch Nginx upstream back to Angular

   ```bash
   # On the Nginx server
   sed -i 's/nextjs-app:3000/angular-app:80/' /etc/nginx/nginx.conf
   nginx -s reload
   ```

2. **DNS-level rollback (< 15 min):** Point DNS to Angular deployment
3. **Data considerations:** Both apps use the same Fineract backend, so no data migration is needed for rollback
4. **Session handling:** Angular and Next.js use different session mechanisms. Users will need to re-authenticate after rollback.

### 7.5 Cutover Checklist

- [ ] All modules migrated and tested
- [ ] E2E tests passing against Next.js in staging
- [ ] Performance benchmarks match or exceed Angular
- [ ] Error rate monitoring configured
- [ ] Rollback procedure documented and tested
- [ ] Hash URL redirects verified
- [ ] Translation completeness verified for all 13 locales
- [ ] External integrations (reports, bulk import) verified
- [ ] Stakeholder sign-off obtained
- [ ] Maintenance window scheduled
- [ ] Communication sent to users

---

## 8. Security Considerations

### 8.1 Content Security Policy (CSP)

```typescript
// next.config.ts
const cspHeader = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' blob: data:;
  font-src 'self';
  connect-src 'self' ${process.env.NEXT_PUBLIC_FINERACT_API_URL};
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
`;

const nextConfig = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: cspHeader.replace(/\n/g, '')
          },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' }
        ]
      }
    ];
  }
};
```

### 8.2 CORS Configuration

CORS is handled at the Nginx level for the Fineract API proxy. The Next.js app itself does not need CORS headers since it serves its own API routes.

For the Next.js API routes (if any):

```typescript
// app/api/[...route]/route.ts
import { NextResponse } from 'next/server';

const ALLOWED_ORIGINS = [
  process.env.NEXT_PUBLIC_APP_URL
].filter(Boolean);

export async function OPTIONS(request: Request) {
  const origin = request.headers.get('origin');
  if (origin && ALLOWED_ORIGINS.includes(origin)) {
    return new NextResponse(null, {
      status: 204,
      headers: {
        'Access-Control-Allow-Origin': origin,
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
        'Access-Control-Max-Age': '86400'
      }
    });
  }
  return new NextResponse(null, { status: 403 });
}
```

### 8.3 API Key and Secret Management

| Secret                    | Storage                         | Access Pattern         |
| ------------------------- | ------------------------------- | ---------------------- |
| `AUTH_SECRET`             | Environment variable            | Server-side only       |
| `OIDC_CLIENT_SECRET`      | Environment variable            | Server-side only       |
| `SENTRY_AUTH_TOKEN`       | CI/CD secrets                   | Build-time only        |
| Fineract user credentials | User session (encrypted cookie) | Per-request            |
| Fineract API base auth    | Environment variable            | Server-side proxy only |

Important:

- Never expose Fineract credentials to the browser
- Use server-side API routes or Server Components to proxy authenticated requests
- Rotate `AUTH_SECRET` periodically
- Use strong, unique secrets per environment

### 8.4 Request Validation

```typescript
// middleware.ts -- additional security checks
export function middleware(request: NextRequest) {
  // Block suspicious paths
  const blockedPaths = [
    '/wp-admin',
    '/wp-login',
    '/.env',
    '/phpinfo',
    '/actuator',
    '/debug',
    '/console'
  ];

  if (blockedPaths.some((p) => request.nextUrl.pathname.startsWith(p))) {
    return new NextResponse(null, { status: 404 });
  }

  // ... auth checks
}
```
