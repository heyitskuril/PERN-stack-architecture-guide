# 2. Project Structure

## The Core Decision: Feature-Based vs Layer-Based

Before laying out a folder structure, you need to make one architectural decision that affects everything else: do you organize code by technical layer, or by feature?

**Layer-based** (also called technical-role organization) groups files by what they are:

```
src/
  controllers/
  services/
  repositories/
  models/
  middlewares/
  utils/
```

**Feature-based** (also called vertical slice organization) groups files by what they belong to:

```
src/
  features/
    users/
    orders/
    products/
  shared/
  infrastructure/
```

Layer-based is familiar and easy to understand for a single developer on a small project. The problem is that it does not scale — as the codebase grows, every feature touches every folder, making it hard to understand the blast radius of a change, hard to reason about what a feature does, and hard to move a feature if the project ever splits.

Feature-based organization keeps everything related to a feature together. When you are working on `orders`, you only need to look inside `features/orders/`. The trade-off is that shared code (auth, logging, database client) needs to live somewhere sensible outside of features.

**This guide recommends feature-based organization for most PERN projects above toy-app scale.**

---

## Monorepo vs Separate Repositories

For a standard PERN application, the backend and frontend are separate applications. You have two choices for how to manage them:

**Option A — Monorepo (recommended for most teams):**

A single repository containing both the frontend and backend. This makes it easy to share TypeScript types between client and server, manage dependencies consistently, run CI on both in one place, and keep version history unified.

Tools that support this well: npm workspaces (built in), pnpm workspaces, Turborepo for caching and task orchestration.

**Option B — Separate repositories:**

Backend in one repo, frontend in another. This works well when teams are large enough that the frontend and backend are owned by different groups, or when the deployment pipelines are fundamentally different. For most small-to-medium teams, this adds friction without meaningful benefit.

---

## Recommended Monorepo Structure

```
project-root/
├── apps/
│   ├── web/                    # React frontend (Vite + React)
│   └── server/                 # Express backend
├── packages/
│   └── shared/                 # Shared TypeScript types, validation schemas, utilities
├── .github/
│   └── workflows/
├── .env.example
├── package.json                # Root workspace config
├── turbo.json                  # (if using Turborepo)
├── tsconfig.base.json
└── README.md
```

---

## Backend Structure (apps/server)

```
apps/server/
├── src/
│   ├── features/
│   │   ├── users/
│   │   │   ├── users.router.ts
│   │   │   ├── users.service.ts
│   │   │   ├── users.repository.ts
│   │   │   ├── users.schema.ts         # Zod validation schemas
│   │   │   ├── users.types.ts
│   │   │   └── users.test.ts
│   │   ├── auth/
│   │   │   ├── auth.router.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.repository.ts
│   │   │   ├── auth.schema.ts
│   │   │   └── auth.test.ts
│   │   └── [other-features]/
│   ├── shared/
│   │   ├── middleware/
│   │   │   ├── authenticate.ts
│   │   │   ├── authorize.ts
│   │   │   ├── error-handler.ts
│   │   │   └── validate.ts
│   │   ├── errors/
│   │   │   ├── AppError.ts
│   │   │   └── error-codes.ts
│   │   └── utils/
│   │       ├── logger.ts
│   │       └── pagination.ts
│   ├── infrastructure/
│   │   ├── database/
│   │   │   └── prisma.ts               # Prisma client singleton
│   │   └── cache/
│   │       └── redis.ts                # Redis client (if applicable)
│   ├── config/
│   │   └── env.ts                      # Validated environment config
│   └── app.ts                          # Express app setup (no server.listen here)
├── server.ts                           # Entry point — only calls app.listen
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── tsconfig.json
├── .env.example
└── package.json
```

### Why Split `app.ts` and `server.ts`

`app.ts` creates and exports the Express application. `server.ts` starts the HTTP server. This separation allows you to import `app` in tests without starting the server, which is the standard pattern for testing Express applications with Supertest.

---

## Frontend Structure (apps/web)

```
apps/web/
├── src/
│   ├── features/
│   │   ├── users/
│   │   │   ├── components/
│   │   │   │   ├── UserCard.tsx
│   │   │   │   └── UserList.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useUsers.ts
│   │   │   ├── api/
│   │   │   │   └── users.api.ts
│   │   │   └── pages/
│   │   │       ├── UsersPage.tsx
│   │   │       └── UserDetailPage.tsx
│   │   └── [other-features]/
│   ├── shared/
│   │   ├── components/
│   │   │   ├── ui/                     # Primitive UI components (Button, Input, etc.)
│   │   │   └── layout/                 # Layout components (Header, Sidebar, etc.)
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   └── utils/
│   │       └── format.ts
│   ├── lib/
│   │   ├── api-client.ts               # Axios or fetch wrapper
│   │   └── query-client.ts             # TanStack Query client
│   ├── router/
│   │   └── index.tsx                   # React Router route definitions
│   ├── providers/
│   │   └── AppProviders.tsx            # Wraps app with all context providers
│   └── main.tsx                        # Entry point
├── index.html
├── vite.config.ts
├── tsconfig.json
└── package.json
```

---

## Shared Package (packages/shared)

```
packages/shared/
├── src/
│   ├── types/
│   │   ├── user.types.ts
│   │   ├── auth.types.ts
│   │   └── api.types.ts                # API response wrappers, pagination types
│   └── schemas/
│       ├── user.schema.ts              # Zod schemas used on both client and server
│       └── auth.schema.ts
├── tsconfig.json
└── package.json
```

Sharing Zod schemas between the frontend and backend means that validation logic stays in one place. When the backend changes a field, the frontend's form validation automatically reflects it.

---

## What Not to Put in the Repo

- `.env` files — use `.env.example` instead
- Build output directories (`dist/`, `build/`)
- Editor configuration that is specific to one person's setup (`.vscode/settings.json` is borderline — a shared `extensions.json` is acceptable but `settings.json` is not)

---

## Sources

- Singer, Ryan. *Shape Up*. Basecamp, 2019. https://basecamp.com/shapeup — Sections on vertical slicing and scope.
- Martin, Robert C. *Clean Architecture: A Craftsman's Guide to Software Structure and Design*. Prentice Hall, 2017. — The dependency rule and folder organization around use cases.
- Osmani, Addy. "Patterns for Large-Scale JavaScript Application Architecture." 2012. https://addyosmani.com/largescalejavascript/ — Principles that still apply in a component-based world.
- Turborepo Documentation. https://turbo.build/repo/docs — Monorepo tooling reference.
