# 4. PostgreSQL Design Patterns

## Why PostgreSQL

PostgreSQL is not the default choice here because it is trendy. It is the default because it is correct for the vast majority of web applications: it enforces data integrity at the database level, supports complex queries, handles concurrent writes correctly with MVCC (Multi-Version Concurrency Control), and has been in active development for over 35 years.

The persistence layer is the hardest to change after launch. Choosing a database that compromises on correctness for the sake of initial simplicity is a trade-off that compounds over time.

---

## Schema Design Principles

### Normalize first, denormalize deliberately

Start with a normalized schema (3NF as the baseline). Denormalize only when you have a measured performance problem that normalization causes. Premature denormalization embeds assumptions about query patterns into the schema — assumptions that are often wrong and expensive to undo.

**First Normal Form (1NF):** Every column stores one atomic value. No comma-separated lists in a column. No repeated column groups.

**Second Normal Form (2NF):** Every non-key column depends on the full primary key, not just part of it. This matters when you have composite primary keys.

**Third Normal Form (3NF):** Every non-key column depends only on the primary key, not on another non-key column. If a column can be derived from another non-key column, it belongs in a separate table.

### Use UUIDs or ULIDs for primary keys

Sequential integer IDs are predictable. An attacker who knows that `GET /orders/100` works can trivially try `GET /orders/99`, `/orders/101`, and so on. Even with authorization checks in place, sequential IDs leak information about the volume of your data.

UUIDs (`gen_random_uuid()` in PostgreSQL 13+) are non-sequential and safe to expose. ULIDs (Universally Unique Lexicographically Sortable Identifiers) add the benefit of being sortable by creation time, which matters for indexed queries on large tables.

Prisma configuration for UUIDs:

```prisma
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### Always include audit timestamps

Every table that represents a business entity should have `created_at` and `updated_at` at minimum. Prisma handles `updatedAt` automatically with the `@updatedAt` attribute.

For sensitive entities — anything involving money, access control, or user data — also include `created_by` and consider a separate audit log table that records who changed what and when.

### Soft deletes for recoverable data

Hard deleting rows makes data recovery impossible and breaks foreign key relationships in ways that can cascade. For most business entities, use a `deleted_at` timestamp column instead.

```prisma
model Order {
  id        String    @id @default(uuid())
  userId    String
  status    OrderStatus
  deletedAt DateTime?

  @@index([deletedAt])
}
```

In your repository, filter out soft-deleted records as the default behavior:

```typescript
async findAll(): Promise<Order[]> {
  return prisma.order.findMany({
    where: { deletedAt: null },
  });
}
```

---

## Indexing Strategy

Indexes speed up reads and slow down writes. Index deliberately, not preemptively.

**Always index:**
- Primary keys (automatic)
- Foreign keys (not automatic in PostgreSQL — you must create these explicitly, or Prisma handles them)
- Columns used in `WHERE` clauses that are frequently queried
- Columns used in `ORDER BY` on large tables

**Consider indexing:**
- Columns used in `JOIN` conditions
- Composite indexes for queries that filter on multiple columns frequently together

**Do not index:**
- Columns with very low cardinality (boolean columns, status columns with 2-3 values) — the query planner often ignores these indexes anyway
- Every column as a default — excessive indexes degrade write performance

Example in Prisma schema:

```prisma
model Order {
  id        String      @id @default(uuid())
  userId    String
  status    OrderStatus
  createdAt DateTime    @default(now())
  deletedAt DateTime?

  user      User        @relation(fields: [userId], references: [id])

  @@index([userId])                        // Foreign key index
  @@index([status])                        // Frequently filtered column
  @@index([userId, status])                // Composite for common combined query
  @@index([deletedAt])                     // For soft-delete filtering
}
```

---

## Migration Strategy

### Never apply migrations manually in production

Migrations applied by hand are not reproducible, not auditable, and dangerous under pressure. All migrations must run through the deployment pipeline.

### Migration workflow with Prisma

```bash
# 1. Modify schema.prisma
# 2. Generate and apply migration in development
npx prisma migrate dev --name describe_the_change

# 3. In production CI/CD pipeline
npx prisma migrate deploy
```

`prisma migrate dev` generates a SQL migration file and applies it. `prisma migrate deploy` applies pending migrations without generating new ones — this is the command for production.

### Write backwards-compatible migrations

During a deployment, there is a window where the new code and the old schema (or new schema and old code) coexist. Migrations that are not backwards-compatible can cause downtime.

The safe pattern for adding a required column:

```sql
-- Step 1: Add as nullable (deploy this first)
ALTER TABLE orders ADD COLUMN notes TEXT;

-- Step 2: Backfill existing rows
UPDATE orders SET notes = '' WHERE notes IS NULL;

-- Step 3: Add NOT NULL constraint (deploy after backfill is complete)
ALTER TABLE orders ALTER COLUMN notes SET NOT NULL;
```

In Prisma, this means sometimes writing the migration in multiple deploy cycles rather than one.

### Always write a rollback plan

For any non-trivial migration, know what you will do if the deployment fails and you need to go back. A new column can be dropped. A renamed column requires more careful handling. A dropped column cannot be undone from a migration alone — it requires a data restore.

---

## Query Patterns with Prisma

### Avoid N+1 queries

N+1 is the most common performance problem in ORM-based applications. It occurs when you fetch a list of records, then loop over them and issue a separate query for each one.

```typescript
// BAD: N+1 — fetches all orders, then issues one query per order for its user
const orders = await prisma.order.findMany();
for (const order of orders) {
  const user = await prisma.user.findUnique({ where: { id: order.userId } }); // N queries
}

// GOOD: Single query with include
const orders = await prisma.order.findMany({
  include: { user: true },
});
```

### Paginate all list queries

Never return unbounded lists from a database query in production. Every `findMany` that is exposed through an API must have pagination.

Cursor-based pagination is recommended for large or frequently updated datasets:

```typescript
async findByUserId(userId: string, cursor?: string, limit = 20): Promise<Order[]> {
  return prisma.order.findMany({
    where: { userId, deletedAt: null },
    take: limit,
    skip: cursor ? 1 : 0,             // Skip the cursor record itself
    cursor: cursor ? { id: cursor } : undefined,
    orderBy: { createdAt: 'desc' },
  });
}
```

Offset-based pagination is simpler but has a known problem: as pages change (inserts, deletes), the offset drifts and users see duplicated or skipped records. Use offset pagination only for datasets that do not change frequently.

### Select only the columns you need

Prisma returns all columns by default. For queries on large tables or in hot code paths, use `select` to fetch only the fields needed:

```typescript
const userSummaries = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    name: true,
    // Does not fetch password hash, internal flags, large text fields, etc.
  },
});
```

### Use transactions for operations that must succeed together

If two writes must both succeed or both fail, wrap them in a transaction:

```typescript
async transferCredit(fromUserId: string, toUserId: string, amount: number): Promise<void> {
  await prisma.$transaction(async (tx) => {
    await tx.user.update({
      where: { id: fromUserId },
      data: { creditBalance: { decrement: amount } },
    });

    await tx.user.update({
      where: { id: toUserId },
      data: { creditBalance: { increment: amount } },
    });

    await tx.creditTransfer.create({
      data: { fromUserId, toUserId, amount },
    });
  });
}
```

---

## Connection Pooling

Node.js applications should not open a new database connection per request. Prisma uses connection pooling internally, but the behavior varies depending on how you deploy:

- **Single Node.js process:** Prisma's built-in pool is sufficient.
- **Serverless / many instances:** Use PgBouncer or Prisma Accelerate to pool connections at the infrastructure level. PostgreSQL has a hard limit on concurrent connections, and serverless functions that each hold their own pool will exhaust it.

Instantiate the Prisma client as a singleton to avoid creating multiple pools:

```typescript
// infrastructure/database/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma || new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
});

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

The `globalForPrisma` pattern prevents Next.js and other hot-reloading environments from creating a new client on every module reload in development.

---

## Sources

- Kleppmann, Martin. *Designing Data-Intensive Applications*. O'Reilly Media, 2017. — MVCC, transactions, consistency guarantees.
- Hernandez, Michael J. *Database Design for Mere Mortals*, 4th ed. Addison-Wesley, 2020. — Normalization, entity design, relationship modeling.
- Karwin, Bill. *SQL Antipatterns: Avoiding the Pitfalls of Database Programming*. Pragmatic Bookshelf, 2010. — Chapter on naive trees, polymorphic associations, and poor indexing.
- Braintree Engineering. "Safe Operations for High-Volume PostgreSQL." https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/ — Zero-downtime migration techniques.
- PostgreSQL Documentation. "Indexes." https://www.postgresql.org/docs/current/indexes.html
- Prisma Documentation. "Connection Management." https://www.prisma.io/docs/guides/performance-and-optimization/connection-management
