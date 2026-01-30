
# Convex Demos

## Welcome to Convex

**Convex is the reactive backend-as-a-service for web developers.**

Convex makes it dead simple to store your app's shared state and sync it
everywhere. As a developer, you interact with your shared state using JavaScript
or TypeScript functions and bind that state to your reactive app.

Realtime updates are automatic. Every user everywhere sees the current version
of your state.

Convex is fully serverless and automatically handles caching and scaling.

**Get started at [docs.convex.dev](https://docs.convex.dev)!**

This repo contains demo apps to get you started with the Convex platform.

---

## Using Claude Code with This Repo

This repo has **33 self-contained demo projects**, each in its own directory
with independent `package.json` and `convex/` folders. There is no monorepo
tooling — navigate into a specific project to work with it.

### Key Convex Patterns (with reference demos)

**Schema & Validation** — Define your data model in `convex/schema.ts` using
`defineSchema` and `defineTable` with validators from `convex/values`:

- `relational-data-modeling/convex/schema.ts` — tables, `v.id()` references, secondary indexes
- `args-validation/convex/messages.ts` — arg validators and return type validators on functions
- `zod-validation-ts/convex/messages.ts` — alternative Zod-based validation with `convex-helpers`

**Queries, Mutations, Actions** — Backend logic lives in `convex/*.ts` files,
exported as named functions using builders from `convex/_generated/server`:

- `query({args, handler})` — read-only, reactive, called via `useQuery` on the client
- `mutation({args, handler})` — read-write, transactional, called via `useMutation`
- `action({args, handler})` — for side effects (HTTP fetches, external APIs); cannot access db directly, uses `ctx.runMutation`/`ctx.runQuery` to interact with data
- `internalMutation` / `internalQuery` — not exposed to client, callable via `internal.*` references

See `tutorial/convex/messages.ts` for the simplest example.
See `giphy-action/convex/messages.ts` for action + internalMutation pattern.

**React Client Integration** — Wrap your app in `<ConvexProvider>`, then use hooks:

```tsx
import { useQuery, useMutation } from "convex/react";
import { api } from "../convex/_generated/api";

const messages = useQuery(api.messages.list);
const send = useMutation(api.messages.send);
```

See `tutorial/src/main.tsx` (provider setup) and `tutorial/src/App.tsx` (hooks usage).

**Search & Vector Search:**

- `search/convex/schema.ts` — `.searchIndex()` for full-text search
- `vector-search/convex/schema.ts` — `.vectorIndex()` for embeddings with filter fields

**File Storage:**

- `file-storage/convex/messages.ts` — `ctx.storage.generateUploadUrl()` and `ctx.storage.getUrl()`

**Scheduling & Cron Jobs:**

- `scheduling/convex/messages.ts` — `ctx.scheduler.runAfter()` for delayed execution
- `cron-jobs/convex/crons.ts` — `cronJobs()` with interval/cron scheduling

**HTTP Endpoints:**

- `http/convex/http.ts` — `httpRouter()` with path and prefix routing

**Pagination:**

- `pagination/convex/messages.ts` — `paginationOptsValidator` and `.paginate()`

**Custom Errors:**

- `custom-errors/convex/messages.ts` — `ConvexError` for structured error data

**Testing:**

- `convex-test/convex/posts.test.ts` — `convex-test` + vitest for testing queries/mutations

**Auth (Clerk):**

- `users-and-clerk/` — Clerk integration
- `users-and-clerk-webhooks/` — Clerk webhook-based user sync

### Project Structure Convention

Each demo follows this structure:

```
project-name/
├── convex/
│   ├── _generated/    # Auto-generated types (don't edit)
│   ├── schema.ts      # Database schema (optional but recommended)
│   ├── tsconfig.json  # Convex-specific TS config
│   └── *.ts           # Your query/mutation/action files
├── src/               # Frontend (React+Vite projects)
│   ├── main.tsx       # ConvexProvider setup
│   └── App.tsx        # UI with useQuery/useMutation hooks
├── package.json
└── vite.config.mts
```

### Dev Workflow

```sh
cd <project-name>
npm install
npm run dev   # runs convex dev + vite dev in parallel
```

The `dev` script uses `npm-run-all --parallel` to start both the Convex backend
(`convex dev` — watches for function changes and pushes to your dev deployment)
and the frontend dev server simultaneously.

---

## Inline Convex Documentation (`docs/`)

The `docs/` directory contains a full copy of the official Convex documentation
source (from the [convex-backend](https://github.com/get-convex/convex-backend)
monorepo). This makes the repo a **self-contained reference** — Claude Code can
read both working demo code and authoritative docs without leaving the repo or
fetching URLs at runtime.

### What's in `docs/`

The copy mirrors
[`npm-packages/docs`](https://github.com/get-convex/convex-backend/tree/main/npm-packages/docs)
from the upstream repo. Key content lives in `docs/docs/`:

| Path | Covers |
|------|--------|
| `docs/docs/functions/` | Queries, mutations, actions, internal functions, HTTP actions |
| `docs/docs/database/` | Tables, schemas, indexes, document IDs, data types |
| `docs/docs/auth/` | Authentication setup, Clerk, Auth0, custom providers |
| `docs/docs/client/` | React hooks, ConvexProvider, optimistic updates |
| `docs/docs/file-storage/` | Upload, download, serving files |
| `docs/docs/search/` | Full-text search indexes |
| `docs/docs/scheduling/` | Cron jobs, scheduled functions |
| `docs/docs/ai/` | AI/LLM integration patterns |
| `docs/docs/agents/` | AI agent patterns with Convex |
| `docs/docs/testing/` | Testing Convex functions with vitest |
| `docs/docs/production/` | Deployment, environment variables, logging |
| `docs/docs/quickstart/` | Framework-specific quickstart guides |
| `docs/docs/tutorial/` | Step-by-step tutorial |
| `docs/docs/components/` | Convex components system |

The directory also includes the Docusaurus site scaffolding (`docusaurus.config.ts`,
`sidebars.js`, `src/`, `static/`) and config files, but the `.mdx` and `.md`
files under `docs/docs/` are the primary reference material.

### How this copy was created

A Git sparse checkout was used to pull only the `npm-packages/docs` subtree
from the upstream repo (commit `e205886`):

```sh
git clone --depth 1 --filter=blob:none --sparse \
  https://github.com/get-convex/convex-backend.git /tmp/convex-backend
cd /tmp/convex-backend
git sparse-checkout set npm-packages/docs
cp -r npm-packages/docs <this-repo>/docs
rm -rf /tmp/convex-backend
```

### Updating to the latest version

Re-run the same commands to replace `docs/` with the latest upstream content:

```sh
rm -rf docs
git clone --depth 1 --filter=blob:none --sparse \
  https://github.com/get-convex/convex-backend.git /tmp/convex-backend
cd /tmp/convex-backend
git sparse-checkout set npm-packages/docs
cp -r npm-packages/docs "$(dirs -l +1)/docs"
rm -rf /tmp/convex-backend
```

This uses `--depth 1` (single commit) and `--filter=blob:none` with sparse
checkout so only the docs subtree is fetched — not the entire convex-backend
repo.

---

## Inline `convex-helpers` Library (`convex-helpers/`)

The `convex-helpers/` directory contains a full copy of the
[convex-helpers](https://github.com/get-convex/convex-helpers) repository — the
community-standard utility library for Convex. This gives Claude Code direct
access to every helper's source, documentation, and usage examples without
needing to fetch from npm or GitHub at runtime.

### Directory map

The library source lives in `convex-helpers/packages/convex-helpers/`. The
detailed README with full usage docs is at
`convex-helpers/packages/convex-helpers/README.md`.

#### Server helpers (`convex-helpers/packages/convex-helpers/server/`)

| File | What it does |
|------|-------------|
| `customFunctions.ts` | Build custom `query`/`mutation`/`action` builders with auth, middleware, ctx extensions |
| `relationships.ts` | Relationship helpers: many-to-many, one-to-many traversal |
| `migrations.ts` | Stateful data migrations that run incrementally |
| `retries.ts` | Action retry wrapper with exponential backoff |
| `rateLimit.ts` | Rate limiting (token bucket and fixed window) |
| `sessions.ts` | Server-side session tracking via client-generated session IDs |
| `rowLevelSecurity.ts` | Row-level security (RLS) — wrap `db` with access rules |
| `zod.ts` | Zod validation for Convex function args (Zod 3 legacy) |
| `zod3.ts` | Zod 3 validation support |
| `zod4.ts` | Zod 4 (Mini) validation support |
| `hono.ts` | Hono framework integration for HTTP endpoints |
| `crud.ts` | Auto-generate CRUD functions for a table |
| `filter.ts` | Filter database queries with arbitrary JS predicates |
| `pagination.ts` | Manual pagination and `paginator` helper |
| `stream.ts` | Composable `QueryStream`s — merge, join, filter across indexes |
| `triggers.ts` | Database triggers — run logic on document create/update/delete |
| `cors.ts` | CORS support for `HttpRouter` |
| `compare.ts` | Deep comparison utilities for Convex values |

#### Client/React helpers

| File | What it does |
|------|-------------|
| `packages/convex-helpers/react/cache/` | `ConvexQueryCacheProvider` — client-side query caching |
| `packages/convex-helpers/react/sessions.ts` | React hooks for session tracking (`useSessionQuery`, etc.) |
| `packages/convex-helpers/react.ts` | Re-export entry point for React helpers |

#### Top-level modules

| File | What it does |
|------|-------------|
| `packages/convex-helpers/index.ts` | Core utilities (e.g. `pruneNull`, `asyncMap`, `getOrThrow`) |
| `packages/convex-helpers/validators.ts` | Validator utilities (`partial`, `pick`, `omit`, `deprecated`, `brandedString`, `literals`, etc.) |
| `packages/convex-helpers/standardSchema.ts` | Standard Schema adapter for Convex validators |
| `packages/convex-helpers/testing.ts` | Testing utilities for local backend tests |
| `packages/convex-helpers/browser.ts` | Browser-safe utilities |

#### CLI tools

| File | What it does |
|------|-------------|
| `packages/convex-helpers/cli/tsApiSpec.ts` | TypeScript API type generation |
| `packages/convex-helpers/cli/openApiSpec.ts` | OpenAPI spec generation from Convex functions |

#### Usage examples (`convex-helpers/convex/`)

| File | Demonstrates |
|------|-------------|
| `schema.ts` | Schema used by all examples |
| `rlsExample.ts` | Row-level security setup |
| `zodFunctionsExample.ts` | Zod-validated Convex functions |
| `migrationsExample.ts` | Stateful migrations |
| `relationshipsExample.ts` | Relationship traversal helpers |
| `retriesExample.ts` | Action retries |
| `sessionsExample.ts` | Session tracking |
| `triggersExample.ts` | Database triggers |
| `streamsExample.ts` | Composable query streams |
| `presence.ts` | Presence tracking functions |
| `http.ts` | HTTP endpoint with Hono + CORS |

#### Client-side hooks (`convex-helpers/src/hooks/`)

| File | What it does |
|------|-------------|
| `usePresence.ts` | Presence tracking React hook |
| `useTypingIndicator.ts` | Typing indicator built on presence |
| `useSingleFlight.ts` | Throttle client requests by single-flighting |
| `useStableQuery.ts` | Return stale results during parameter changes |
| `useLatestValue.ts` | Always-current value ref utility |

### How this copy was created

A shallow clone of the full repo (commit `22df7f7`):

```sh
git clone --depth 1 https://github.com/get-convex/convex-helpers.git /tmp/convex-helpers
cp -r /tmp/convex-helpers convex-helpers
rm -rf convex-helpers/.git convex-helpers/.github convex-helpers/package-lock.json
rm -rf /tmp/convex-helpers
```

### Updating to the latest version

Re-run the same commands to replace `convex-helpers/` with the latest upstream:

```sh
rm -rf convex-helpers
git clone --depth 1 https://github.com/get-convex/convex-helpers.git /tmp/convex-helpers
cp -r /tmp/convex-helpers convex-helpers
rm -rf convex-helpers/.git convex-helpers/.github convex-helpers/package-lock.json
rm -rf /tmp/convex-helpers
```

This clones only the latest commit (`--depth 1`) so the fetch is fast.

---

## Inline Convex Components (`inlined-components/`)

The `inlined-components/` directory contains full copies of official
[Convex components](https://convex.dev/components) — self-contained backend
modules that bundle functions, schemas, and isolated data. This gives Claude
Code direct access to each component's source, schema, client API, and usage
examples.

See [`inlined-components/README.md`](inlined-components/README.md) for the
full directory map, per-component API guides, and sync instructions.

### Components included

| Directory | NPM package | What it does |
|-----------|-------------|-------------|
| `inlined-components/stripe/` | `@convex-dev/stripe` | Stripe payments, subscriptions, billing, and webhook sync |
| `inlined-components/workos-authkit/` | `@convex-dev/workos-authkit` | WorkOS AuthKit user sync, event handlers, and auth actions |
| `inlined-components/workpool/` | `@convex-dev/workpool` | Async work pools with parallelism limits, retries, and completion callbacks |

### How these copies were created

Each repo was shallow-cloned (`--depth 1`) and copied in with `.git`, `.github`,
and lock files removed. The same pattern used for `convex-helpers/`:

```sh
git clone --depth 1 https://github.com/get-convex/stripe.git /tmp/convex-stripe
cp -r /tmp/convex-stripe inlined-components/stripe
rm -rf inlined-components/stripe/.git inlined-components/stripe/.github \
       inlined-components/stripe/package-lock.json
rm -rf /tmp/convex-stripe
```

### Updating to the latest versions

A sync script in `inlined-components/README.md` refreshes all components at
once. To update a single component, re-run its clone + copy + cleanup commands.
