# 9. Testing Strategy

## Why Testing Matters in a PERN Context

The IBM Systems Sciences Institute research that is widely cited in the industry found that the cost to fix a bug discovered in production is significantly higher than catching it during development — sometimes cited as 100x, depending on the study. The numbers vary by source, but the direction is consistent: bugs found earlier are cheaper to fix.

For a PERN application specifically, testing also provides a different kind of value: it enforces the layer boundaries described in the architecture section. If you try to write a unit test for a service that has Prisma calls embedded directly in it, the test is extremely difficult to write. That difficulty is a signal from the tests telling you the code is structured incorrectly. Tests make architecture problems visible.

---

## The Test Pyramid

The test pyramid describes the relative proportion of test types. The base is large (many unit tests), the middle is medium (integration tests), and the top is small (end-to-end tests).

```
       [E2E]
     [Integration]
  [Unit Tests - most]
```

The reason for this shape: unit tests are fast, cheap to write, and give immediate feedback. E2E tests are slow, expensive, and brittle. You want many of the former and few of the latter.

---

## Backend Testing

### Unit Tests (Service Layer)

The service layer is the primary target for unit tests. Because the service layer does not import Express or Prisma, you can test it by injecting mock repositories.

The recommended approach in this codebase is **constructor injection** — repositories are passed into the service rather than instantiated inside it. This makes swapping mocks straightforward.

```typescript
// features/orders/orders.service.ts (adjusted for testability)
export class OrdersService {
  constructor(
    private ordersRepository: OrdersRepository,
    private usersRepository: UsersRepository,
  ) {}

  // ... methods
}
```

```typescript
// features/orders/orders.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { OrdersService } from './orders.service';
import { AppError } from '../../shared/errors/AppError';

const mockOrdersRepository = {
  create: vi.fn(),
  findById: vi.fn(),
  findByUserId: vi.fn(),
};

const mockUsersRepository = {
  findById: vi.fn(),
  findByEmail: vi.fn(),
  create: vi.fn(),
};

describe('OrdersService', () => {
  let service: OrdersService;

  beforeEach(() => {
    vi.clearAllMocks();
    service = new OrdersService(
      mockOrdersRepository as any,
      mockUsersRepository as any,
    );
  });

  describe('createOrder', () => {
    it('throws USER_NOT_FOUND when user does not exist', async () => {
      mockUsersRepository.findById.mockResolvedValue(null);

      await expect(
        service.createOrder('user-id', { items: [{ productId: 'p1', quantity: 1, price: 100 }] })
      ).rejects.toMatchObject({
        code: 'USER_NOT_FOUND',
        statusCode: 404,
      });
    });

    it('throws ACCOUNT_NOT_VERIFIED when user account is not verified', async () => {
      mockUsersRepository.findById.mockResolvedValue({ id: 'user-id', isVerified: false });

      await expect(
        service.createOrder('user-id', { items: [{ productId: 'p1', quantity: 1, price: 100 }] })
      ).rejects.toMatchObject({
        code: 'ACCOUNT_NOT_VERIFIED',
        statusCode: 403,
      });
    });

    it('throws INVALID_ORDER when items array is empty', async () => {
      mockUsersRepository.findById.mockResolvedValue({ id: 'user-id', isVerified: true });

      await expect(
        service.createOrder('user-id', { items: [] })
      ).rejects.toMatchObject({
        code: 'INVALID_ORDER',
        statusCode: 400,
      });
    });

    it('creates an order and returns it when input is valid', async () => {
      const mockUser = { id: 'user-id', isVerified: true };
      const mockOrder = { id: 'order-id', userId: 'user-id', total: 200, items: [] };

      mockUsersRepository.findById.mockResolvedValue(mockUser);
      mockOrdersRepository.create.mockResolvedValue(mockOrder);

      const result = await service.createOrder('user-id', {
        items: [{ productId: 'p1', quantity: 2, price: 100 }],
      });

      expect(result).toEqual(mockOrder);
      expect(mockOrdersRepository.create).toHaveBeenCalledWith(
        expect.objectContaining({ userId: 'user-id', total: 200 })
      );
    });
  });
});
```

### Integration Tests (API Endpoints)

Integration tests verify that routes, middleware, services, and repositories work together correctly. They use a real test database (not mocks) to ensure the actual SQL queries work.

Use `supertest` to make HTTP requests against the Express app:

```typescript
// features/auth/auth.router.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import request from 'supertest';
import { createApp } from '../../app';
import { prisma } from '../../infrastructure/database/prisma';

const app = createApp();

beforeAll(async () => {
  // Run migrations on test database
  await prisma.$executeRaw`TRUNCATE TABLE "User" RESTART IDENTITY CASCADE`;
});

afterAll(async () => {
  await prisma.$disconnect();
});

describe('POST /api/v1/auth/register', () => {
  it('returns 201 and sets auth cookies on valid registration', async () => {
    const response = await request(app)
      .post('/api/v1/auth/register')
      .send({ email: 'test@example.com', password: 'securepassword123', name: 'Test User' });

    expect(response.status).toBe(201);
    expect(response.body.status).toBe('success');

    // Cookies should be set (httpOnly, so not visible in response body)
    const cookies = response.headers['set-cookie'];
    expect(cookies).toBeDefined();
    expect(cookies.some((c: string) => c.startsWith('accessToken'))).toBe(true);
  });

  it('returns 409 when email is already registered', async () => {
    // Create user first
    await request(app)
      .post('/api/v1/auth/register')
      .send({ email: 'existing@example.com', password: 'securepassword123', name: 'Existing' });

    // Try to register with the same email
    const response = await request(app)
      .post('/api/v1/auth/register')
      .send({ email: 'existing@example.com', password: 'anotherpassword', name: 'Duplicate' });

    expect(response.status).toBe(409);
    expect(response.body.code).toBe('EMAIL_IN_USE');
  });

  it('returns 400 when request body is invalid', async () => {
    const response = await request(app)
      .post('/api/v1/auth/register')
      .send({ email: 'not-an-email', password: '123' });

    expect(response.status).toBe(400);
    expect(response.body.code).toBe('VALIDATION_ERROR');
  });
});
```

---

## Frontend Testing

### Unit Tests (Hooks and Utilities)

Test custom hooks and utility functions. TanStack Query hooks can be tested with the `@testing-library/react` wrapper:

```typescript
// features/orders/hooks/useOrders.test.ts
import { describe, it, expect, vi } from 'vitest';
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useOrders } from './useOrders';
import { ordersApi } from '../api/orders.api';
import { ReactNode } from 'react';

vi.mock('../api/orders.api');

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return ({ children }: { children: ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
};

describe('useOrders', () => {
  it('returns orders on successful fetch', async () => {
    const mockOrders = [{ id: '1', total: 100 }];
    vi.mocked(ordersApi.getAll).mockResolvedValue(mockOrders as any);

    const { result } = renderHook(() => useOrders(), { wrapper: createWrapper() });

    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data).toEqual(mockOrders);
  });
});
```

### Component Tests

Test components with `@testing-library/react`. Test behavior, not implementation details.

```typescript
// features/orders/components/OrderCard.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { OrderCard } from './OrderCard';

const mockOrder = {
  id: 'order-1',
  status: 'confirmed' as const,
  total: 250000,
  createdAt: new Date('2024-01-15'),
  items: [{ id: 'item-1', productId: 'p1', quantity: 2, price: 125000 }],
};

describe('OrderCard', () => {
  it('displays the order total', () => {
    render(<OrderCard order={mockOrder} />);
    expect(screen.getByText(/250,000/)).toBeInTheDocument();
  });

  it('displays the order status', () => {
    render(<OrderCard order={mockOrder} />);
    expect(screen.getByText(/confirmed/i)).toBeInTheDocument();
  });
});
```

---

## Test Configuration

### Vitest Setup

```typescript
// vitest.config.ts (backend)
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      include: ['src/features/**/*.ts'],
      exclude: ['src/features/**/*.router.ts', 'src/features/**/*.types.ts'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 70,
      },
    },
  },
});
```

```typescript
// vitest.config.ts (frontend)
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
  },
});
```

```typescript
// src/test/setup.ts (frontend)
import '@testing-library/jest-dom';
```

---

## Coverage Targets

Coverage as a percentage is a proxy metric, not a goal. The goal is confidence that the code works. That said, targets are useful for enforcing a minimum:

- Service layer (business logic): 80%+ line and function coverage
- API routes: covered by integration tests, not individual unit tests
- Utility functions: 90%+ (they are pure functions and easy to test completely)
- UI components: focus on critical and complex components, not visual layout

Do not chase 100% coverage. Time spent testing `console.log` statements and straightforward getters is time not spent testing meaningful business logic.

---

## Sources

- Khorikov, Vladimir. *Unit Testing: Principles, Practices, and Patterns*. Manning Publications, 2020. — Especially chapters 1–4 on what makes a good unit test.
- Freeman, Steve, and Nat Pryce. *Growing Object-Oriented Software, Guided by Tests*. Addison-Wesley, 2009. — Outside-in TDD methodology.
- Kent C. Dodds. "Testing Implementation Details." https://kentcdodds.com/blog/testing-implementation-details
- Vitest Documentation. https://vitest.dev
- Testing Library Documentation. https://testing-library.com/docs/react-testing-library/intro/
- Supertest Documentation. https://github.com/ladjs/supertest
