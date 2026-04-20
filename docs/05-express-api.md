# 5. Express API Layer

## Setting Up the Application

The Express application setup should happen in one place, `app.ts`, separate from where the server starts (`server.ts`). This keeps the HTTP server out of your test setup.

```typescript
// src/app.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { errorHandler } from './shared/middleware/error-handler';
import { requestLogger } from './shared/middleware/request-logger';
import usersRouter from './features/users/users.router';
import authRouter from './features/auth/auth.router';
import ordersRouter from './features/orders/orders.router';

export function createApp() {
  const app = express();

  // Security headers
  app.use(helmet());

  // CORS — allowlist known origins, never use wildcard in production
  app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [],
    credentials: true,
  }));

  // Body parsing
  app.use(express.json({ limit: '10kb' }));

  // Request logging
  app.use(requestLogger);

  // Health check — no auth required
  app.get('/health', (req, res) => {
    res.status(200).json({ status: 'ok' });
  });

  // API routes
  app.use('/api/v1/auth', authRouter);
  app.use('/api/v1/users', usersRouter);
  app.use('/api/v1/orders', ordersRouter);

  // 404 handler — must come after all routes
  app.use((req, res) => {
    res.status(404).json({
      status: 'error',
      code: 'NOT_FOUND',
      message: `Route ${req.method} ${req.path} does not exist.`,
    });
  });

  // Global error handler — must be last and must have 4 parameters
  app.use(errorHandler);

  return app;
}
```

```typescript
// server.ts
import { createApp } from './src/app';
import { env } from './src/config/env';
import { logger } from './src/shared/utils/logger';

const app = createApp();

const server = app.listen(env.PORT, () => {
  logger.info(`Server running on port ${env.PORT} in ${env.NODE_ENV} mode`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  logger.info('SIGTERM received. Shutting down gracefully...');
  server.close(() => {
    logger.info('HTTP server closed.');
    process.exit(0);
  });
});
```

---

## Router Structure

Each feature owns its router. The router file should be thin — it maps HTTP verbs and paths to handler functions, applies middleware, and passes control to the service layer.

```typescript
// features/users/users.router.ts
import { Router } from 'express';
import { authenticate } from '../../shared/middleware/authenticate';
import { authorize } from '../../shared/middleware/authorize';
import { validate } from '../../shared/middleware/validate';
import { updateUserSchema } from './users.schema';
import { UsersService } from './users.service';

const router = Router();
const usersService = new UsersService();

router.get(
  '/me',
  authenticate,
  async (req, res, next) => {
    try {
      const user = await usersService.getUserById(req.user.id);
      res.status(200).json({ status: 'success', data: user });
    } catch (error) {
      next(error);
    }
  }
);

router.patch(
  '/me',
  authenticate,
  validate(updateUserSchema),
  async (req, res, next) => {
    try {
      const user = await usersService.updateUser(req.user.id, req.body);
      res.status(200).json({ status: 'success', data: user });
    } catch (error) {
      next(error);
    }
  }
);

router.get(
  '/',
  authenticate,
  authorize(['admin']),
  async (req, res, next) => {
    try {
      const users = await usersService.getAllUsers();
      res.status(200).json({ status: 'success', data: users });
    } catch (error) {
      next(error);
    }
  }
);

export default router;
```

---

## Request Validation Middleware

Validate every incoming request before it reaches the service layer. Use Zod for schema definition — it integrates cleanly with TypeScript and gives you type inference from your schemas.

```typescript
// shared/middleware/validate.ts
import { Request, Response, NextFunction } from 'express';
import { ZodSchema, ZodError } from 'zod';
import { AppError } from '../errors/AppError';

export function validate(schema: ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      // ZodError gives you field-level error messages
      const formattedErrors = result.error.errors.map((e) => ({
        field: e.path.join('.'),
        message: e.message,
      }));

      return res.status(400).json({
        status: 'error',
        code: 'VALIDATION_ERROR',
        message: 'Request validation failed.',
        details: formattedErrors,
      });
    }

    // Replace req.body with the parsed (and potentially transformed) value
    req.body = result.data;
    next();
  };
}
```

```typescript
// features/users/users.schema.ts
import { z } from 'zod';

export const updateUserSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  email: z.string().email().optional(),
}).refine(
  (data) => Object.keys(data).length > 0,
  { message: 'At least one field must be provided.' }
);

export type UpdateUserDto = z.infer<typeof updateUserSchema>;
```

Using `z.infer<typeof schema>` means your DTO types are always in sync with your validation schema. If you change the schema, the types follow automatically.

---

## Authentication Middleware

```typescript
// shared/middleware/authenticate.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { env } from '../../config/env';
import { AppError } from '../errors/AppError';

// Extend Express Request type globally
declare global {
  namespace Express {
    interface Request {
      user: {
        id: string;
        role: string;
      };
    }
  }
}

export function authenticate(req: Request, res: Response, next: NextFunction) {
  const token = req.cookies?.accessToken;

  if (!token) {
    return next(new AppError('UNAUTHORIZED', 'Authentication required.', 401));
  }

  try {
    const payload = jwt.verify(token, env.JWT_ACCESS_SECRET) as { sub: string; role: string };
    req.user = { id: payload.sub, role: payload.role };
    next();
  } catch (error) {
    return next(new AppError('UNAUTHORIZED', 'Invalid or expired token.', 401));
  }
}
```

---

## Authorization Middleware

```typescript
// shared/middleware/authorize.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../errors/AppError';

export function authorize(allowedRoles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return next(new AppError('UNAUTHORIZED', 'Authentication required.', 401));
    }

    if (!allowedRoles.includes(req.user.role)) {
      return next(new AppError('FORBIDDEN', 'Insufficient permissions.', 403));
    }

    next();
  };
}
```

---

## Consistent API Response Shape

Every response from the API — success or error — should follow the same structure. This makes client-side handling predictable.

```typescript
// Success
{
  "status": "success",
  "data": { ... }
}

// Success with pagination
{
  "status": "success",
  "data": [ ... ],
  "meta": {
    "total": 100,
    "nextCursor": "abc123"
  }
}

// Error
{
  "status": "error",
  "code": "ORDER_NOT_FOUND",
  "message": "The requested order was not found."
}

// Validation error
{
  "status": "error",
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed.",
  "details": [
    { "field": "email", "message": "Invalid email address." }
  ]
}
```

---

## Rate Limiting

Rate limit authentication endpoints and all public-facing endpoints. The `express-rate-limit` package is the standard choice:

```typescript
import rateLimit from 'express-rate-limit';

export const authRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10,                   // 10 attempts per window
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    status: 'error',
    code: 'TOO_MANY_REQUESTS',
    message: 'Too many attempts. Please try again in 15 minutes.',
  },
});

export const apiRateLimiter = rateLimit({
  windowMs: 60 * 1000,       // 1 minute
  max: 100,                   // 100 requests per minute per IP
  standardHeaders: true,
  legacyHeaders: false,
});
```

Apply `apiRateLimiter` globally and `authRateLimiter` specifically to login and registration routes.

---

## Environment Configuration

Validate environment variables at startup. If a required variable is missing, the application should refuse to start — failing loudly is better than failing mysteriously later.

```typescript
// config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_ACCESS_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  ALLOWED_ORIGINS: z.string().optional(),
});

const result = envSchema.safeParse(process.env);

if (!result.success) {
  console.error('Invalid environment configuration:');
  console.error(result.error.flatten().fieldErrors);
  process.exit(1);
}

export const env = result.data;
```

---

## Sources

- Express.js Documentation. https://expressjs.com/en/4x/api.html
- Zod Documentation. https://zod.dev
- OWASP. "REST Security Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
- Lauret, Arnaud. *The Design of Web APIs*. Manning Publications, 2019. — API consistency and response shape design.
- The 12-Factor App. "Config." https://12factor.net/config — Environment-based configuration.
- Helmet.js Documentation. https://helmetjs.github.io/
