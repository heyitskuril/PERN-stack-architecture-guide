<div align="center">

# PERN Stack Architecture Guide

**A practical, production-oriented reference for building full-stack web applications with PostgreSQL, Express, React, and Node.js.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](./CONTRIBUTING.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](./CONTRIBUTING.md)
![Last Updated](https://img.shields.io/badge/last%20updated-2026-blue)

</div>

---

This guide is not a tutorial. It does not walk you through building a to-do app. It is a structured reference covering the architectural decisions, folder structures, layering conventions, and engineering standards that separate a weekend project from something you can actually maintain, scale, and hand off to another developer.

Every recommendation here is backed by industry-recognized sources — books, specifications, and engineering literature that have stood up to scrutiny. Where trade-offs exist, they are stated explicitly.

---

## Who This Is For

| Audience | How to use this |
|----------|-----------------|
| Full-stack developers working in or migrating to the PERN stack | Follow the sections in order — each one builds on the last |
| Developers who have built PERN apps before but want a more structured approach | Jump to the sections most relevant to what you are building right now |
| Teams looking to establish shared conventions across a codebase | Use individual documents as the basis for internal engineering standards |

---

## What Is Covered

| # | Section | Summary |
|---|---------|---------|
| 1 | [Stack Overview](./docs/01-stack-overview.md) | Why PERN, when it fits, and when it does not |
| 2 | [Project Structure](./docs/02-project-structure.md) | Folder organization, feature-based vs layer-based, monorepo options |
| 3 | [Clean Architecture in PERN](./docs/03-clean-architecture.md) | Layering: presentation, service, repository, domain |
| 4 | [PostgreSQL Design Patterns](./docs/04-postgresql.md) | Schema design, indexing, migrations, query patterns with Prisma |
| 5 | [Express API Layer](./docs/05-express-api.md) | Router structure, middleware, error handling, request validation |
| 6 | [Authentication & Authorization](./docs/06-auth.md) | JWT, session, RBAC, httpOnly cookies, refresh token rotation |
| 7 | [React Architecture](./docs/07-react-architecture.md) | Component design, state management, data fetching patterns |
| 8 | [Environment & Configuration](./docs/08-environment.md) | Config management, secrets, environment parity |
| 9 | [Testing Strategy](./docs/09-testing.md) | Unit, integration, and E2E testing in the PERN context |
| 10 | [Deployment & CI/CD](./docs/10-deployment.md) | CI pipeline, environments, zero-downtime deployment |
| 11 | [Coding Standards](./docs/11-coding-standards.md) | Naming, commits, TypeScript configuration, tooling |
| 12 | [Reference Library](./docs/12-references.md) | All books and resources cited throughout this guide |

Each section includes working code examples, the reasoning behind each decision, and explicit trade-offs — not just "do this."

---

## Tech Stack Covered

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=flat-square&logo=express&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=react&logoColor=black)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)

---

## How to Use This

**Read linearly the first time.** Sections are ordered deliberately — folder structure decisions in Section 2 inform the clean architecture patterns in Section 3, which inform how the Express layer is set up in Section 5. Reading out of order works, but some context will be missing.

**Use as a reference afterward.** Once you are familiar with the guide, individual documents work as standalone references when making specific decisions on a project: "how should I structure my migrations," "what is the right place to put authorization logic," "what do I actually need to test."

**Clone it and adapt it.** Fork this repo and modify the sections that do not fit your context. The structure and reasoning matter more than any specific recommendation.

---

## Companion Repositories

This guide is part of a series on engineering and product development:

- [Product Development Playbook](https://github.com/heyitskuril/product-development-playbook) — The complete 17-phase guide from idea to launch, covering product discovery, system architecture, API design, security, deployment, and post-launch iteration

---

## References & Standards

This guide draws from:

- Books: *Clean Architecture* (Martin), *Designing Data-Intensive Applications* (Kleppmann), *The Pragmatic Programmer* (Hunt & Thomas), and 20+ others — all cited inline
- Standards: OWASP Top 10, NIST SP 800-63B, RFC 8725 (JWT), The 12-Factor App
- Engineering literature: DORA research, Google SRE Book, Braintree Engineering, Google Engineering Practices

The full list is in [Section 12 — Reference Library](./docs/12-references.md).

---

## Contributing

Found a broken link? Know a better reference? Want to add a missing section?

See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines. All contributions that improve accuracy, clarity, or coverage are welcome.

---

## License

MIT License — free to use, adapt, and distribute. See [LICENSE](./LICENSE) for details.

---

## About the author

<div align="center">

<img src="https://avatars.githubusercontent.com/heyitskuril" width="80" style="border-radius: 50%;" />

**Kuril**

A dreamer… who still working on it to make it real.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/heyitskuril)
[![Instagram](https://img.shields.io/badge/Instagram-E4405F?style=flat-square&logo=instagram&logoColor=white)](https://instagram.com/heyitskuril)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/heyitskuril)
[![Email](https://img.shields.io/badge/Email-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:kuril.dev@gmail.com)

</div>

---

<div align="center">

If this guide helped you, consider giving it a ⭐ — it helps others find it too.

</div>
