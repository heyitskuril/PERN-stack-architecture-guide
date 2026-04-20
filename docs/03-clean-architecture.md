# 3. Clean Architecture in PERN

## Why Layering Matters

The core problem with most Express applications that grow beyond a few hundred lines is that business logic ends up scattered everywhere. Logic lives in route handlers. Route handlers call the database directly. Validation happens in some places but not others. When you need to change how a pricing rule works, you cannot find it in one place.

This is not a PERN-specific problem. It is the natural consequence of building without explicit architectural layers.

Clean Architecture, as described by Robert C. Martin, and the layered patterns described earlier by Martin Fowler in *Patterns of Enterprise Application Architecture*, both make the same fundamental argument: **separate what the application does from how it does it**. Business rules should not know about HTTP. Database queries should not know about business rules. The direction of dependency should flow inward toward the domain, not outward toward infrastructure.

Applied to a PERN backend, this means four explicit layers.

---

## The Four Layers

### Layer 1: Router (Presentation / Delivery)

The router layer is responsible for one thing: handling HTTP. It receives a request, extracts and validates the input, calls the appropriate service, and returns a response. It knows about HTTP status codes and request/response shapes. It does not know about the database. It does not contain business logic.

```typescript
// features/orders/orders.router.ts
import { Router } from 'express';
import { authenticate } from '../../shared/middleware/authenticate';
import { validate } from '../../shared/middleware/validate';
import { createOrderSchema } from './orders.schema';
import { OrdersService } from './orders.service';

const router = Router();
const ordersService = new OrdersService();

router.post('/', authenticate, validate(createOrderSchema), async (req, res, next) => {
  try {
    const order = await ordersService.createOrder(req.user.id, req.body);
    res.status(201).json({ status: 'success', data: order });
  } catch (error) {
    next(error);
  }
});

router.get('/:id', authenticate, async (req, res, next) => {
  try {
    const order = await ordersService.getOrderById(req.params.id, req.user.id);
    res.status(200).json({ status: 'success', data: order });
  } catch (error) {
    next(error);
  }
});

export default router;
```

**What belongs here:** HTTP method and path, input extraction (`req.body`, `req.params`, `req.query`), validation middleware, response formatting, error passing to `next`.

**What does not belong here:** database queries, business calculations, sending emails, any logic that would need to be tested without starting an HTTP server.

---

### Layer 2: Service (Application / Business Logic)

The service layer is where business logic lives. It orchestrates operations: it calls repositories to fetch data, applies rules, makes decisions, and coordinates side effects (sending a notification, publishing an event, writing to a log).

The service layer does not know about Express. It does not import `Request` or `Response`. It receives plain arguments and returns plain values. This is what makes it independently testable.

```typescript
// features/orders/orders.service.ts
import { OrdersRepository } from './orders.repository';
import { UsersRepository } from '../users/users.repository';
import { AppError } from '../../shared/errors/AppError';
import { CreateOrderDto, Order } from './orders.types';

export class OrdersService {
  private ordersRepository: OrdersRepository;
  private usersRepository: UsersRepository;

  constructor() {
    this.ordersRepository = new OrdersRepository();
    this.usersRepository = new UsersRepository();
  }

  async createOrder(userId: string, dto: CreateOrderDto): Promise<Order> {
    const user = await this.usersRepository.findById(userId);

    if (!user) {
      throw new AppError('USER_NOT_FOUND', 'User not found.', 404);
    }

    if (!user.isVerified) {
      throw new AppError('ACCOUNT_NOT_VERIFIED', 'Account must be verified before placing an order.', 403);
    }

    // Business rule: orders cannot be empty
    if (!dto.items || dto.items.length === 0) {
      throw new AppError('INVALID_ORDER', 'An order must contain at least one item.', 400);
    }

    const total = this.calculateTotal(dto.items);
    const order = await this.ordersRepository.create({ ...dto, userId, total });

    return order;
  }

  async getOrderById(orderId: string, requestingUserId: string): Promise<Order> {
    const order = await this.ordersRepository.findById(orderId);

    if (!order) {
      throw new AppError('ORDER_NOT_FOUND', 'Order not found.', 404);
    }

    // Authorization check — users can only see their own orders
    if (order.userId !== requestingUserId) {
      throw new AppError('FORBIDDEN', 'You do not have access to this order.', 403);
    }

    return order;
  }

  private calculateTotal(items: CreateOrderDto['items']): number {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}
```

**What belongs here:** business rules, authorization checks, orchestrating multiple repository calls, calculations, error throwing with domain-specific error codes.

**What does not belong here:** Prisma calls, SQL queries, HTTP-specific logic.

---

### Layer 3: Repository (Data Access)

The repository layer is responsible for all database interaction. Its methods speak the language of the domain, not the language of the database. From the outside, you call `ordersRepository.findById(id)` — you do not care whether that is a SQL query, a Prisma call, or a Redis lookup.

This layer is the boundary between your application and your persistence mechanism. If you ever switch from Prisma to a raw SQL client, you change the repository layer and nothing else.

```typescript
// features/orders/orders.repository.ts
import { prisma } from '../../infrastructure/database/prisma';
import { Order, CreateOrderInput } from './orders.types';

export class OrdersRepository {
  async create(data: CreateOrderInput): Promise<Order> {
    return prisma.order.create({
      data: {
        userId: data.userId,
        total: data.total,
        status: 'pending',
        items: {
          create: data.items.map((item) => ({
            productId: item.productId,
            quantity: item.quantity,
            price: item.price,
          })),
        },
      },
      include: { items: true },
    });
  }

  async findById(id: string): Promise<Order | null> {
    return prisma.order.findUnique({
      where: { id },
      include: { items: true },
    });
  }

  async findByUserId(userId: string): Promise<Order[]> {
    return prisma.order.findMany({
      where: { userId },
      include: { items: true },
      orderBy: { createdAt: 'desc' },
    });
  }
}
```

**What belongs here:** Prisma queries, database-specific logic, result mapping if the raw database shape differs from what the service expects.

**What does not belong here:** business rules, HTTP, error formatting.

---

### Layer 4: Domain (Types and Entities)

This layer holds the types that describe your domain entities. In TypeScript projects using Prisma, this is often partially generated from the Prisma schema. You extend or compose from that as needed.

```typescript
// features/orders/orders.types.ts

export interface OrderItem {
  id: string;
  productId: string;
  quantity: number;
  price: number;
}

export interface Order {
  id: string;
  userId: string;
  status: 'pending' | 'confirmed' | 'processing' | 'completed' | 'cancelled';
  total: number;
  items: OrderItem[];
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateOrderDto {
  items: {
    productId: string;
    quantity: number;
    price: number;
  }[];
}

export type CreateOrderInput = CreateOrderDto & { userId: string; total: number };
```

---

## The Dependency Rule

The key rule is that dependencies only point inward. Routers depend on services. Services depend on repositories. Repositories depend on the database client. Nothing in the inner layers (service, domain) imports from outer layers (router, infrastructure).

```
Router → Service → Repository → Database
         ↑
    (also imports domain types)
```

If you find yourself importing from `express` inside a service file, that is a signal the boundary has been crossed incorrectly.

---

## Error Handling Across Layers

Define a single `AppError` class that all layers use to communicate errors. The outer error handler middleware translates these into HTTP responses.

```typescript
// shared/errors/AppError.ts
export class AppError extends Error {
  public readonly code: string;
  public readonly statusCode: number;
  public readonly isOperational: boolean;

  constructor(code: string, message: string, statusCode: number = 500) {
    super(message);
    this.code = code;
    this.statusCode = statusCode;
    this.isOperational = true;

    // Captures the correct stack trace in V8 (Node.js / Chrome)
    Error.captureStackTrace(this, this.constructor);
  }
}
```

```typescript
// shared/middleware/error-handler.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../errors/AppError';
import { logger } from '../utils/logger';

export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err instanceof AppError && err.isOperational) {
    return res.status(err.statusCode).json({
      status: 'error',
      code: err.code,
      message: err.message,
    });
  }

  // Unexpected (programmer) errors — log the full error, return generic message
  logger.error({ err, req: { method: req.method, url: req.url } }, 'Unexpected error');

  return res.status(500).json({
    status: 'error',
    code: 'INTERNAL_SERVER_ERROR',
    message: 'An unexpected error occurred.',
  });
}
```

This distinction between operational errors (expected, user-facing) and programmer errors (unexpected, internal) is taken directly from Joyent's Node.js error handling best practices and remains the standard approach.

---

## Sources

- Martin, Robert C. *Clean Architecture: A Craftsman's Guide to Software Structure and Design*. Prentice Hall, 2017. — The dependency rule, screaming architecture, use cases.
- Fowler, Martin. *Patterns of Enterprise Application Architecture*. Addison-Wesley, 2002. — Service Layer pattern (p. 133), Repository pattern (p. 322), Transaction Script pattern.
- Evans, Eric. *Domain-Driven Design: Tackling Complexity in the Heart of Software*. Addison-Wesley, 2003. — Repositories, entities, domain events.
- Joyent. "Error Handling in Node.js." https://www.joyent.com/node-js/production/design/errors — The operational vs programmer error distinction.
- Prisma Documentation. https://www.prisma.io/docs
