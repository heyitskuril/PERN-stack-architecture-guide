# 1. Stack Overview

## What PERN Is

PERN is an acronym for four technologies that together cover the full web application stack:

- **PostgreSQL** — Relational database
- **Express** — HTTP framework for Node.js
- **React** — UI library for the browser
- **Node.js** — JavaScript runtime for the server

These four are not a framework in the sense that Rails or Laravel is a framework. There is no official PERN scaffold, no generator that sets up conventions for you, and no enforced structure. That is both its strength and its risk. You get flexibility, but you also get the responsibility of making architecture decisions that a more opinionated framework would make for you.

This guide exists to make those decisions deliberately and consistently.

---

## Why This Stack Makes Sense

**One language across the full stack.** JavaScript (or TypeScript) runs on both the client and the server. This reduces context-switching, allows code sharing (types, validation schemas, utility functions) between frontend and backend, and makes it easier for a small team to move across layers.

**PostgreSQL is a serious database.** Unlike some stacks that default to MongoDB for convenience, PostgreSQL gives you ACID transactions, foreign key constraints, complex joins, full-text search, JSON column support, and a 35-year track record. It is used in production by companies of every size. The argument that PostgreSQL does not scale is not supported by evidence — Shopify, Instagram in its early years, and GitHub have all run on PostgreSQL at significant scale.

**React has the largest ecosystem.** In 2024, React remains the most widely adopted frontend library. This means more hiring options, more libraries, more community answers, and more long-term support than most alternatives.

**Express is minimal and well-understood.** It does not enforce a structure, which means you need to bring your own. That is exactly what this guide provides. Express is also extremely stable — its core API has not changed significantly in years.

---

## When PERN Is a Good Fit

- Standard web applications: dashboards, SaaS products, internal tools, marketplaces
- Teams comfortable with JavaScript/TypeScript across the stack
- Projects where you want control over your architecture without buying into a full-stack framework's conventions
- Applications that need a relational data model (foreign keys, joins, transactions)

## When to Consider Alternatives

- **You need server-side rendering out of the box** — Consider Next.js on the frontend while keeping the Express backend, or move to a full Next.js setup with its API routes. React alone is a client-side library.
- **You are building a heavily real-time application** — Express can handle WebSockets, but something like Fastify or dedicated real-time infrastructure may serve you better.
- **Your data is fundamentally document-based with no relational structure** — PostgreSQL can handle JSON columns, but if your entire data model is documents, a document database may be a better fit.
- **You need a full-stack framework with conventions already decided** — Consider Next.js, Remix, or a non-JavaScript alternative like Rails or Laravel if you prefer convention over configuration.

---

## TypeScript as a Default

This guide assumes TypeScript throughout. The reason is practical: TypeScript catches a significant class of errors at compile time that would otherwise only appear at runtime. At scale — or even at small scale when working across a codebase you do not have entirely in your head — this matters.

The TypeScript compiler is not an optional layer on top of JavaScript. Treat it as part of the stack. Enable strict mode. Do not use `any` unless you have a documented reason.

The configuration section covers the specific TypeScript settings this guide recommends.

---

## Sources

- Fowler, Martin. *Patterns of Enterprise Application Architecture*. Addison-Wesley, 2002. — Chapter on layering and the rationale for separating concerns across the stack.
- PostgreSQL Global Development Group. *PostgreSQL 16 Documentation*. https://www.postgresql.org/docs/16/
- Node.js Foundation. *Node.js Documentation*. https://nodejs.org/en/docs/
- TypeScript Team. *TypeScript Handbook*. https://www.typescriptlang.org/docs/
