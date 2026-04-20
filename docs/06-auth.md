# 6. Authentication & Authorization

## Choosing an Auth Strategy

Two patterns are common in PERN applications:

**Session-based (stateful):** The server creates a session record on login. The client stores a session ID in a cookie. On each request, the server looks up the session. Sessions can be immediately invalidated server-side because the server is the source of truth.

**JWT-based (stateless):** On login, the server signs a token containing claims (user ID, role) and returns it. The client sends it on every request. The server validates the signature but holds no state. The trade-off is that tokens cannot be truly invalidated before expiry — a revoked token is still valid until it expires.

For most PERN web applications, **a hybrid approach** is recommended: short-lived JWTs (15 minutes) for access tokens stored in `httpOnly` cookies, combined with long-lived refresh tokens (7–30 days) also in `httpOnly` cookies, with the refresh token stored in the database so it can be revoked.

This gives you stateless request authentication with real revocation capability.

---

## Why httpOnly Cookies, Not localStorage

Storing tokens in `localStorage` is convenient but exposes them to XSS attacks. Any JavaScript on the page — including injected scripts — can read `localStorage`. An `httpOnly` cookie cannot be accessed by JavaScript at all. It is sent automatically by the browser on every request and is only readable by the server.

This point is not a matter of preference. OWASP's Authentication Cheat Sheet and the IETF RFC 8725 (JWT Best Current Practices) both explicitly recommend `httpOnly` cookies over `localStorage` for token storage.

---

## Implementation

### Auth Service

```typescript
// features/auth/auth.service.ts
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import { UsersRepository } from '../users/users.repository';
import { AuthRepository } from './auth.repository';
import { AppError } from '../../shared/errors/AppError';
import { env } from '../../config/env';
import { RegisterDto, LoginDto, AuthTokens } from './auth.types';

const SALT_ROUNDS = 12;
const ACCESS_TOKEN_EXPIRY = '15m';
const REFRESH_TOKEN_EXPIRY = '7d';

export class AuthService {
  private usersRepository: UsersRepository;
  private authRepository: AuthRepository;

  constructor() {
    this.usersRepository = new UsersRepository();
    this.authRepository = new AuthRepository();
  }

  async register(dto: RegisterDto): Promise<AuthTokens> {
    const existingUser = await this.usersRepository.findByEmail(dto.email);

    if (existingUser) {
      throw new AppError('EMAIL_IN_USE', 'An account with this email already exists.', 409);
    }

    const passwordHash = await bcrypt.hash(dto.password, SALT_ROUNDS);
    const user = await this.usersRepository.create({
      email: dto.email,
      name: dto.name,
      passwordHash,
    });

    return this.generateTokens(user.id, user.role);
  }

  async login(dto: LoginDto): Promise<AuthTokens> {
    const user = await this.usersRepository.findByEmail(dto.email);

    if (!user) {
      // Use the same error message for both "user not found" and "wrong password"
      // to prevent user enumeration attacks
      throw new AppError('INVALID_CREDENTIALS', 'Invalid email or password.', 401);
    }

    const passwordMatch = await bcrypt.compare(dto.password, user.passwordHash);

    if (!passwordMatch) {
      throw new AppError('INVALID_CREDENTIALS', 'Invalid email or password.', 401);
    }

    return this.generateTokens(user.id, user.role);
  }

  async refresh(refreshToken: string): Promise<AuthTokens> {
    let payload: { sub: string; role: string };

    try {
      payload = jwt.verify(refreshToken, env.JWT_REFRESH_SECRET) as typeof payload;
    } catch {
      throw new AppError('UNAUTHORIZED', 'Invalid or expired refresh token.', 401);
    }

    // Check that the token exists in the database (i.e., has not been revoked)
    const storedToken = await this.authRepository.findRefreshToken(refreshToken);

    if (!storedToken) {
      // Possible token reuse — invalidate all refresh tokens for this user
      await this.authRepository.revokeAllUserRefreshTokens(payload.sub);
      throw new AppError('UNAUTHORIZED', 'Refresh token reuse detected. Please log in again.', 401);
    }

    // Rotate: delete the used token, issue a new pair
    await this.authRepository.deleteRefreshToken(refreshToken);
    return this.generateTokens(payload.sub, payload.role);
  }

  async logout(refreshToken: string): Promise<void> {
    await this.authRepository.deleteRefreshToken(refreshToken);
  }

  private async generateTokens(userId: string, role: string): Promise<AuthTokens> {
    const accessToken = jwt.sign(
      { sub: userId, role },
      env.JWT_ACCESS_SECRET,
      { expiresIn: ACCESS_TOKEN_EXPIRY }
    );

    const refreshToken = jwt.sign(
      { sub: userId, role },
      env.JWT_REFRESH_SECRET,
      { expiresIn: REFRESH_TOKEN_EXPIRY }
    );

    // Store refresh token in database for revocation capability
    await this.authRepository.saveRefreshToken({
      token: refreshToken,
      userId,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    });

    return { accessToken, refreshToken };
  }
}
```

### Auth Router

```typescript
// features/auth/auth.router.ts
import { Router, CookieOptions } from 'express';
import { validate } from '../../shared/middleware/validate';
import { authenticate } from '../../shared/middleware/authenticate';
import { authRateLimiter } from '../../shared/middleware/rate-limiter';
import { registerSchema, loginSchema } from './auth.schema';
import { AuthService } from './auth.service';

const router = Router();
const authService = new AuthService();

const cookieOptions: CookieOptions = {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production', // HTTPS only in production
  sameSite: 'strict',
  path: '/',
};

const ACCESS_COOKIE_EXPIRY = 15 * 60 * 1000;          // 15 minutes
const REFRESH_COOKIE_EXPIRY = 7 * 24 * 60 * 60 * 1000; // 7 days

router.post('/register', authRateLimiter, validate(registerSchema), async (req, res, next) => {
  try {
    const tokens = await authService.register(req.body);

    res.cookie('accessToken', tokens.accessToken, {
      ...cookieOptions,
      maxAge: ACCESS_COOKIE_EXPIRY,
    });

    res.cookie('refreshToken', tokens.refreshToken, {
      ...cookieOptions,
      maxAge: REFRESH_COOKIE_EXPIRY,
      path: '/api/v1/auth/refresh', // Restrict refresh token cookie to the refresh endpoint only
    });

    res.status(201).json({ status: 'success', message: 'Account created successfully.' });
  } catch (error) {
    next(error);
  }
});

router.post('/login', authRateLimiter, validate(loginSchema), async (req, res, next) => {
  try {
    const tokens = await authService.login(req.body);

    res.cookie('accessToken', tokens.accessToken, {
      ...cookieOptions,
      maxAge: ACCESS_COOKIE_EXPIRY,
    });

    res.cookie('refreshToken', tokens.refreshToken, {
      ...cookieOptions,
      maxAge: REFRESH_COOKIE_EXPIRY,
      path: '/api/v1/auth/refresh',
    });

    res.status(200).json({ status: 'success', message: 'Logged in successfully.' });
  } catch (error) {
    next(error);
  }
});

router.post('/refresh', async (req, res, next) => {
  try {
    const refreshToken = req.cookies?.refreshToken;

    if (!refreshToken) {
      return res.status(401).json({
        status: 'error',
        code: 'UNAUTHORIZED',
        message: 'No refresh token provided.',
      });
    }

    const tokens = await authService.refresh(refreshToken);

    res.cookie('accessToken', tokens.accessToken, {
      ...cookieOptions,
      maxAge: ACCESS_COOKIE_EXPIRY,
    });

    res.cookie('refreshToken', tokens.refreshToken, {
      ...cookieOptions,
      maxAge: REFRESH_COOKIE_EXPIRY,
      path: '/api/v1/auth/refresh',
    });

    res.status(200).json({ status: 'success', message: 'Token refreshed.' });
  } catch (error) {
    next(error);
  }
});

router.post('/logout', authenticate, async (req, res, next) => {
  try {
    const refreshToken = req.cookies?.refreshToken;
    if (refreshToken) {
      await authService.logout(refreshToken);
    }

    res.clearCookie('accessToken');
    res.clearCookie('refreshToken', { path: '/api/v1/auth/refresh' });

    res.status(200).json({ status: 'success', message: 'Logged out successfully.' });
  } catch (error) {
    next(error);
  }
});

export default router;
```

---

## Role-Based Access Control (RBAC)

RBAC is the appropriate model for most applications. Define roles upfront, assign permissions to roles, and assign roles to users. Do not assign permissions directly to individual users — that becomes unmaintainable quickly.

### Permission Matrix Example

| Resource | Action | member | editor | admin |
|----------|--------|--------|--------|-------|
| Own profile | Read | Yes | Yes | Yes |
| Own profile | Update | Yes | Yes | Yes |
| Other profiles | Read | No | No | Yes |
| Articles | Read | Yes | Yes | Yes |
| Articles | Create | No | Yes | Yes |
| Articles | Update (own) | No | Yes | Yes |
| Articles | Delete | No | No | Yes |
| Users | Manage | No | No | Yes |

### Database Schema for Roles

```prisma
enum Role {
  member
  editor
  admin
}

model User {
  id           String   @id @default(uuid())
  email        String   @unique
  passwordHash String
  name         String
  role         Role     @default(member)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}
```

Authorization checks belong in the service layer, not in middleware alone. Middleware can gate access by role, but fine-grained checks (can this user edit this specific resource?) require access to business data and belong in the service.

```typescript
// Service-level authorization check
async updateArticle(articleId: string, requestingUserId: string, role: string, dto: UpdateArticleDto) {
  const article = await this.articlesRepository.findById(articleId);

  if (!article) {
    throw new AppError('NOT_FOUND', 'Article not found.', 404);
  }

  // Admins can edit anything; editors can only edit their own articles
  const canEdit = role === 'admin' || article.authorId === requestingUserId;

  if (!canEdit) {
    throw new AppError('FORBIDDEN', 'You do not have permission to edit this article.', 403);
  }

  return this.articlesRepository.update(articleId, dto);
}
```

---

## Password Rules

Following NIST SP 800-63B (the current US government standard for digital identity):

- Minimum length: 8 characters (12 is a better practical minimum)
- Maximum length: at least 64 characters
- Do not impose complexity rules (uppercase/lowercase/number/symbol requirements) — they reduce entropy rather than increasing it
- Check against known breached passwords using the Have I Been Pwned API (k-Anonymity model — you never send the full password hash)
- Hash with bcrypt at cost factor 12, or Argon2id

```typescript
// Checking against HaveIBeenPwned (k-Anonymity)
import crypto from 'crypto';
import fetch from 'node-fetch';

async function isPasswordBreached(password: string): Promise<boolean> {
  const hash = crypto.createHash('sha1').update(password).digest('hex').toUpperCase();
  const prefix = hash.slice(0, 5);
  const suffix = hash.slice(5);

  const response = await fetch(`https://api.pwnedpasswords.com/range/${prefix}`);
  const text = await response.text();

  return text.split('\n').some((line) => line.startsWith(suffix));
}
```

---

## Sources

- OWASP. "Authentication Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP. "Session Management Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- OWASP. "JSON Web Token Cheat Sheet for Java." https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html — The concepts apply to Node.js.
- IETF. RFC 8725: "JSON Web Token Best Current Practices." https://datatracker.ietf.org/doc/html/rfc8725 — 2020.
- NIST. "Special Publication 800-63B: Digital Identity Guidelines." https://pages.nist.gov/800-63-3/sp800-63b.html
- Richer, Justin, and Antonio Sanso. *OAuth 2 in Action*. Manning Publications, 2017.
- Have I Been Pwned API Documentation. https://haveibeenpwned.com/API/v3
