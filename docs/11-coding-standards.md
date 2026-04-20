# 11. Coding Standards

## Why Standards Matter

The value of a coding standard is not that any one rule is objectively correct. Most naming conventions and formatting rules are arbitrary. The value is consistency — a codebase where every file looks like it was written by the same person is dramatically easier to read, review, and maintain than one where every developer has expressed their own preferences.

Robert C. Martin in *Clean Code* notes that code is read far more often than it is written. Standards are an investment in every future reading.

---

## TypeScript Configuration

Enable strict mode. There is no good reason to disable it in a new project:

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "dist",
    "rootDir": "src",

    // Strict mode
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,

    // Path aliases
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },

    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

The `noUncheckedIndexedAccess` option is worth calling out: it adds `undefined` to the type of any array index access (`array[0]` returns `T | undefined`), which catches a significant class of runtime errors.

---

## Naming Conventions

| Context | Convention | Example |
|---------|-----------|---------|
| Files (general) | `kebab-case` | `order-service.ts` |
| Files (React components) | `PascalCase` | `OrderCard.tsx` |
| Classes | `PascalCase` | `OrdersService` |
| Interfaces and Types | `PascalCase` | `CreateOrderDto` |
| Functions and methods | `camelCase` | `createOrder()` |
| Variables and constants | `camelCase` | `orderTotal` |
| Booleans | prefix with `is`, `has`, `can` | `isVerified`, `hasAccess` |
| Module-level constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_ATTEMPTS` |
| Database tables | `snake_case` (Prisma default) | `order_items` |
| Database columns | `snake_case` | `created_at` |
| Environment variables | `SCREAMING_SNAKE_CASE` | `DATABASE_URL` |
| Zod schemas | suffix with `Schema` | `createOrderSchema` |
| DTO types | suffix with `Dto` | `CreateOrderDto` |
| React hooks | prefix with `use` | `useOrders` |
| React context | suffix with `Context` | `AuthContext` |

---

## ESLint Configuration

```javascript
// eslint.config.mjs
import js from '@eslint/js';
import tsPlugin from '@typescript-eslint/eslint-plugin';
import tsParser from '@typescript-eslint/parser';
import importPlugin from 'eslint-plugin-import';

export default [
  js.configs.recommended,
  {
    files: ['**/*.ts', '**/*.tsx'],
    languageOptions: {
      parser: tsParser,
      parserOptions: {
        project: './tsconfig.json',
      },
    },
    plugins: {
      '@typescript-eslint': tsPlugin,
      import: importPlugin,
    },
    rules: {
      // TypeScript
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/explicit-function-return-type': 'off',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/consistent-type-imports': 'error',

      // Imports
      'import/order': ['error', {
        groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
        'newlines-between': 'always',
      }],
      'import/no-default-export': 'off',

      // General
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'prefer-const': 'error',
      'no-var': 'error',
      'eqeqeq': ['error', 'always'],
    },
  },
];
```

---

## Prettier Configuration

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

---

## Pre-commit Hooks

Use `husky` and `lint-staged` to enforce standards before every commit:

```bash
npm install --save-dev husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yaml,yml}": [
      "prettier --write"
    ]
  }
}
```

```bash
# .husky/pre-commit
npx lint-staged
```

---

## Commit Message Convention

Adopt Conventional Commits. The format is:

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

| Type | When to use |
|------|------------|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation changes only |
| `style` | Formatting, no logic change |
| `refactor` | Code change that is neither a fix nor a feature |
| `test` | Adding or updating tests |
| `chore` | Build process, dependency updates, tooling |
| `perf` | Performance improvement |
| `ci` | CI/CD configuration changes |

Examples:

```
feat(auth): add refresh token rotation

fix(orders): prevent creating orders with empty items array

docs(readme): update local setup instructions

refactor(users): extract email validation to shared utility

test(auth): add integration tests for token refresh endpoint

chore: update prisma to 5.10.0
```

The scope is the feature or module affected. Keep descriptions lowercase and in the imperative mood ("add" not "added", "fix" not "fixes").

Enforce this with `commitlint`:

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```javascript
// commitlint.config.js
export default { extends: ['@commitlint/config-conventional'] };
```

```bash
# .husky/commit-msg
npx --no-install commitlint --edit "$1"
```

---

## Code Review Standards

Every pull request should be reviewed before merging. The review is not about catching bugs alone — it is about knowledge sharing, catching design problems, and maintaining standards.

**As an author:**
- Keep PRs small — under 400 lines of changed code is a reasonable target
- Write a clear PR description: what changed, why, and how to test it
- Link to the relevant issue or task
- Do not open a PR until CI passes

**As a reviewer:**
- Respond within one business day
- Distinguish between blocking issues and non-blocking suggestions (use "nit:" prefix for minor style suggestions)
- Explain the reasoning behind requests — "please change this because..." not just "change this"
- Approve when the code is good enough to ship, not when it is perfect

Google's Engineering Practices guide, which is publicly available, provides the clearest articulation of what to look for in a code review: correctness, design, complexity, tests, naming, comments, style, and documentation.

---

## Sources

- Martin, Robert C. *Clean Code: A Handbook of Agile Software Craftsmanship*. Prentice Hall, 2008.
- Hunt, Andrew, and David Thomas. *The Pragmatic Programmer*, 20th Anniversary Edition. Addison-Wesley, 2019. — DRY, naming, and coding practices.
- Ousterhout, John. *A Philosophy of Software Design*, 2nd ed. Yaknyam Press, 2021. — Deep vs. shallow modules, naming, and comments.
- Conventional Commits Specification. https://www.conventionalcommits.org/
- Google. "Google Engineering Practices." https://google.github.io/eng-practices/
- TypeScript Team. "TypeScript Strict Mode." https://www.typescriptlang.org/tsconfig#strict
