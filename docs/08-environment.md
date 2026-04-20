# 8. Environment & Configuration

## The Core Principle

Configuration that changes between environments — database URLs, API keys, feature flags, allowed origins — must not be hardcoded in source code and must not be committed to version control.

This is Factor III of the Twelve-Factor App methodology: store config in the environment. The test is simple: could you open-source the codebase right now without exposing credentials? If the answer is no, something is in the code that belongs in the environment.

---

## Environment Files

### .env.example (commit this)

The `.env.example` file documents every environment variable the application needs, with placeholder values. Every developer who clones the repo uses this as a template.

```bash
# apps/server/.env.example

# Application
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# JWT Secrets — generate with: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
JWT_ACCESS_SECRET=your-access-secret-here-minimum-32-characters
JWT_REFRESH_SECRET=your-refresh-secret-here-minimum-32-characters

# CORS
ALLOWED_ORIGINS=http://localhost:5173

# Redis (optional)
REDIS_URL=redis://localhost:6379

# Email (optional)
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
```

```bash
# apps/web/.env.example

VITE_API_URL=http://localhost:3000/api/v1
```

### .env (never commit this)

The `.env` file contains actual values for local development. Add it to `.gitignore` before the first commit. A common mistake is creating the file and committing it before adding the gitignore entry — check your git history if you are ever unsure.

```bash
# .gitignore
.env
.env.local
.env.*.local
```

---

## Validated Environment Configuration

Do not use `process.env.SOME_VAR` directly throughout the codebase. Instead, validate and export a typed config object at application startup. If a required variable is missing or malformed, the application exits immediately with a clear error.

This is better than discovering a missing variable at the moment the feature that needs it is called — potentially hours or days after deployment.

```typescript
// apps/server/src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url('DATABASE_URL must be a valid URL'),
  JWT_ACCESS_SECRET: z.string().min(32, 'JWT_ACCESS_SECRET must be at least 32 characters'),
  JWT_REFRESH_SECRET: z.string().min(32, 'JWT_REFRESH_SECRET must be at least 32 characters'),
  ALLOWED_ORIGINS: z.string().optional(),
  REDIS_URL: z.string().url().optional(),
});

const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('\nInvalid environment configuration. Fix the following before starting:\n');
  const errors = parsed.error.flatten().fieldErrors;
  Object.entries(errors).forEach(([field, messages]) => {
    console.error(`  ${field}: ${messages?.join(', ')}`);
  });
  console.error('');
  process.exit(1);
}

export const env = parsed.data;

// Type helper for the rest of the codebase
export type Env = typeof env;
```

On the frontend, Vite exposes environment variables prefixed with `VITE_`. Apply the same validation pattern:

```typescript
// apps/web/src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  VITE_API_URL: z.string().url('VITE_API_URL must be a valid URL'),
});

const parsed = envSchema.safeParse(import.meta.env);

if (!parsed.success) {
  throw new Error(
    `Invalid frontend environment configuration: ${JSON.stringify(parsed.error.flatten().fieldErrors)}`
  );
}

export const env = parsed.data;
```

---

## Environment Parity

The three environments — development, staging, production — should be as similar as possible. Differences between environments are where bugs hide.

Common parity failures:

- Development uses SQLite, production uses PostgreSQL — schema differences and query behavior differences are not caught until deployment
- Development uses in-memory data, production uses Redis — cache invalidation bugs go undetected
- Staging runs with lower resource limits than production — performance problems are invisible until they hit real users

The practical minimum for parity: use the same database engine in development and production (run PostgreSQL locally with Docker). Use the same Node.js version (pin it in `.nvmrc` or use Volta). Use the same environment variable names — only the values differ.

---

## Secret Management in Production

Do not put secrets in your CI/CD configuration as plain text. Use the secret management provided by your platform:

- **GitHub Actions:** GitHub Encrypted Secrets (`Settings > Secrets and variables > Actions`)
- **Vercel:** Environment Variables in the project settings
- **Railway / Render:** Environment variable configuration in the dashboard
- **AWS:** AWS Secrets Manager or Parameter Store

Reference secrets in CI/CD configuration without exposing their values:

```yaml
# .github/workflows/deploy.yml
- name: Deploy to production
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
    JWT_ACCESS_SECRET: ${{ secrets.JWT_ACCESS_SECRET }}
    JWT_REFRESH_SECRET: ${{ secrets.JWT_REFRESH_SECRET }}
  run: npm run deploy
```

---

## Node Version Management

Pin the Node.js version used by the project. Different Node.js versions have different behavior with some APIs. This prevents "works on my machine" problems.

`.nvmrc`:
```
20.11.0
```

Or use Volta for automatic version switching:
```json
// package.json
{
  "volta": {
    "node": "20.11.0",
    "npm": "10.2.4"
  }
}
```

Pin the same version in your CI/CD configuration and in your production deployment.

---

## Sources

- The Twelve-Factor App. "III. Config: Store config in the environment." https://12factor.net/config
- The Twelve-Factor App. "X. Dev/prod parity." https://12factor.net/dev-prod-parity
- Zod Documentation. https://zod.dev
- GitHub. "Using secrets in GitHub Actions." https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions
- Volta — JavaScript Tool Manager. https://volta.sh
