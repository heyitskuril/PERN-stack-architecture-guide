# 10. Deployment & CI/CD

## The Goal

Every change merged to the main branch should be deployable to production within minutes, automatically, without manual steps. If deployment is manual, it is infrequent. If it is infrequent, it is scary. If it is scary, you avoid it. That avoidance is where software delivery slows down and risk accumulates.

DORA (DevOps Research and Assessment) research, published in *Accelerate* (Forsgren, Humble, Kim, 2018), consistently finds that high-performing software teams deploy more frequently — not less. The way to deploy frequently without risk is automation and small, incremental changes.

---

## Environment Strategy

Four environments is the standard:

| Environment | Purpose | Deployed by |
|-------------|---------|-------------|
| `local` | Individual developer machine | Manual `npm run dev` |
| `development` | Shared integration testing after merge to `develop` | CI on push to `develop` |
| `staging` | Production mirror for pre-release validation | CI on merge to `main` |
| `production` | Live system serving users | CD after staging smoke tests pass |

Staging must be as close to production as possible: same database engine, same environment variables (different values), same deployment infrastructure.

---

## GitHub Actions — CI Pipeline

This pipeline runs on every pull request. It must pass before the PR can be merged.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main, develop]

jobs:
  lint-and-type-check:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check (server)
        run: npm run type-check --workspace=apps/server

      - name: Type check (web)
        run: npm run type-check --workspace=apps/web

      - name: Lint
        run: npm run lint

  test-backend:
    name: Backend Tests
    runs-on: ubuntu-latest
    needs: lint-and-type-check

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        env:
          DATABASE_URL: postgresql://testuser:testpassword@localhost:5432/testdb
        run: npx prisma migrate deploy --schema=apps/server/prisma/schema.prisma

      - name: Run unit and integration tests
        env:
          DATABASE_URL: postgresql://testuser:testpassword@localhost:5432/testdb
          JWT_ACCESS_SECRET: test-access-secret-minimum-32-characters-long
          JWT_REFRESH_SECRET: test-refresh-secret-minimum-32-characters-long
          NODE_ENV: test
        run: npm run test:coverage --workspace=apps/server

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: backend-coverage
          path: apps/server/coverage/

  test-frontend:
    name: Frontend Tests
    runs-on: ubuntu-latest
    needs: lint-and-type-check
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm run test:coverage --workspace=apps/web

  build:
    name: Build Verification
    runs-on: ubuntu-latest
    needs: [test-backend, test-frontend]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build
```

---

## GitHub Actions — Deploy Pipeline

This pipeline runs on merge to `main`, deploys to staging, runs smoke tests, then deploys to production.

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run database migrations (staging)
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
        run: npx prisma migrate deploy --schema=apps/server/prisma/schema.prisma

      - name: Deploy to staging
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}   # Replace with your platform
        run: npm run deploy:staging

  smoke-test-staging:
    name: Smoke Tests on Staging
    runs-on: ubuntu-latest
    needs: deploy-staging

    steps:
      - uses: actions/checkout@v4

      - name: Wait for staging to be healthy
        run: |
          for i in {1..10}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" ${{ secrets.STAGING_URL }}/health)
            if [ "$STATUS" = "200" ]; then
              echo "Staging is healthy"
              exit 0
            fi
            echo "Attempt $i: staging returned $STATUS. Waiting..."
            sleep 10
          done
          echo "Staging health check failed after 10 attempts"
          exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: smoke-test-staging
    environment: production   # Requires manual approval in GitHub environment settings

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run database migrations (production)
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
        run: npx prisma migrate deploy --schema=apps/server/prisma/schema.prisma

      - name: Deploy to production
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
        run: npm run deploy:production
```

The `environment: production` setting in GitHub Actions can require a manual approval step before the production deploy runs. This is the recommended gate for most teams — automatic up to staging, manual approval for production.

---

## Database Migrations in CI/CD

Never run migrations manually in production. The pipeline above runs `prisma migrate deploy` as part of the deployment step. This command applies any pending migrations from the `prisma/migrations` folder.

### Migration safety checklist

Before any migration runs in production:
- It has run successfully in development
- It has run successfully in the CI test environment
- It has run successfully in staging
- It is backwards-compatible with the currently deployed code (old code works with new schema during the rollout window)

If a migration is not backwards-compatible (renaming a column, dropping a column, adding a non-nullable column without a default), break it into multiple deployments.

---

## Rollback Strategy

Define the rollback procedure before you need it, not during an incident.

**Application rollback:** Most platforms (Railway, Render, Fly.io) support one-click rollback to a previous deployment. Document which button to press and who has access.

**Database rollback:** This is harder. `prisma migrate deploy` does not automatically roll back. If a migration causes problems:

1. For additive changes (adding columns, adding tables): roll back the application code to the previous version, leave the schema change in place, and clean up the schema in the next planned deployment.
2. For destructive changes (dropping columns): this is why you should not drop columns immediately. Deprecate first, remove after a full deployment cycle confirms the column is not being written to or read from.

The general rule: application rollbacks should be possible within 5 minutes. Schema rollbacks require planning and should be avoided through backwards-compatible migration practices.

---

## Health Check Endpoint

Every deployed service should have a health check endpoint that returns HTTP 200 when the service is running correctly:

```typescript
// Minimal health check
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Extended health check (checks database connectivity)
app.get('/health/detailed', async (req, res) => {
  try {
    await prisma.$queryRaw`SELECT 1`;
    res.status(200).json({ status: 'ok', database: 'connected' });
  } catch (error) {
    res.status(503).json({ status: 'error', database: 'disconnected' });
  }
});
```

The basic `/health` endpoint is for load balancer and uptime monitor checks. The detailed version is for human debugging. Do not expose the detailed version publicly.

---

## Sources

- Forsgren, Nicole, Jez Humble, and Gene Kim. *Accelerate: The Science of Lean Software and DevOps*. IT Revolution, 2018. — DORA metrics, deployment frequency, change failure rate.
- Humble, Jez, and David Farley. *Continuous Delivery: Reliable Software Releases through Build, Test, and Deployment Automation*. Addison-Wesley, 2010.
- Kim, Gene, Jez Humble, Patrick Debois, and John Willis. *The DevOps Handbook*. IT Revolution, 2016.
- Braintree Engineering. "Safe Operations for High-Volume PostgreSQL." https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/
- GitHub Actions Documentation. https://docs.github.com/en/actions
- DORA. "DORA's Software Delivery Metrics." https://dora.dev/
