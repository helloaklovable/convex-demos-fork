# Convex Guardrails Pattern Index

How Convex developers constrain database behavior — computed fields, cascade
deletes, immutability, restricted interfaces, and more — without traditional
database constraints.

## Guarantee Strength Spectrum

From weakest to strongest enforcement:

| Level | Mechanism | Bypassable by | Example |
|-------|-----------|----------------|---------|
| 1. Convention | Explicit wrapper functions | Any code that imports raw `ctx.db` | `createUser(ctx, ...)` |
| 2. Convention + Lint | Wrapper + ESLint `no-restricted-imports` | Disabling lint rule, dashboard, imports | Custom `mutation` from `functions.ts` |
| 3. Runtime interception | Triggers / RLS wrapping `ctx.db` | Forgetting the wrapper, dashboard, imports | `triggers.wrapDB`, `wrapDatabaseWriter` |
| 4. Hard isolation | Component with private tables | Nothing short of modifying the component | `defineComponent()` with restricted API |
| 5. Engine-level | Convex schema validation | Nothing | `v.number()` in `defineTable()` |

---

## Pattern 1: Triggers (Computed Fields, Cascade Deletes, Validation)

**What**: Register functions that auto-run on `ctx.db.insert/patch/replace/delete`.
Atomic with the mutation. Throw to abort.

**Guarantee level**: 3 (runtime interception — only fires through wrapped mutations)

### Computed / Denormalized Fields

| Example | File | Key lines |
|---------|------|-----------|
| `fullName` from `firstName + lastName` | `convex-helpers/packages/convex-helpers/server/triggers.test.ts` | 46-54 |
| Round counter to nearest 10 | `convex-helpers/convex/triggersExample.ts` | 20-32 |
| Denormalized sum of all counters | `convex-helpers/convex/triggersExample.ts` | 50-70 |
| Denormalized count (insert/delete) | `convex-helpers/packages/convex-helpers/server/triggers.test.ts` | 79-91 |
| Aggregate B-tree (count/sum/rank) via `trigger()` | `inlined-components/aggregate/src/client/index.ts` | 862-887 |
| Leaderboard aggregate sync | `inlined-components/aggregate/example/convex/leaderboard.ts` | 46-54 |
| Photo album aggregate sync | `inlined-components/aggregate/example/convex/photos.ts` | 36-45 |

**Most elegant example**: The Aggregate component's `.trigger()` method. You
register it like any other trigger, but internally it maintains a B-tree for
O(log n) count/sum/offset queries. One line to wire up, strong guarantees
within the component boundary:

```ts
triggers.register("leaderboard", aggregateByScore.trigger());
```

### Cascade Deletes

| Example | File | Key lines |
|---------|------|-----------|
| Delete user's messages on user delete | `convex-helpers/packages/convex-helpers/README.md` | 1134-1141 |
| Denormalized count decrement on delete | `convex-helpers/convex/triggersExample.ts` | 65-70 |

### Validation (Abort on Bad Data)

| Example | File | Key lines |
|---------|------|-----------|
| Throw on forbidden value ("The Balrog") | `convex-helpers/packages/convex-helpers/README.md` | ~1097 |
| Email regex validation | Blog post (Triggers article) | N/A (in pasted content) |

### Setup Pattern

Every trigger example follows this structure:

```
convex/functions.ts (or equivalent):
  1. Import raw mutation from _generated/server
  2. Instantiate Triggers<DataModel>()
  3. Register triggers per table
  4. Export wrapped mutation via customMutation(rawMutation, customCtx(triggers.wrapDB))
```

Key files showing this setup:
- `convex-helpers/convex/triggersExample.ts:72-76`
- `inlined-components/aggregate/example/convex/leaderboard.ts:51-54`
- `inlined-components/aggregate/example/convex/photos.ts:40-44`
- `convex-helpers/packages/convex-helpers/server/triggers.test.ts:93`

---

## Pattern 2: Row-Level Security (RLS)

**What**: Wrap `ctx.db` so every read/write is checked against per-table rules.

**Guarantee level**: 3 (runtime interception)

| Example | File | Key lines |
|---------|------|-----------|
| Core implementation (WrapReader, WrapWriter) | `convex-helpers/packages/convex-helpers/server/rowLevelSecurity.ts` | 165-428 |
| Rules type definition | same file | 22-28 |
| Working example with auth rules | `convex-helpers/convex/rlsExample.ts` | 17-83 |
| RLS + Triggers composition | `convex-helpers/convex/triggersExample.ts` | 118-142 |
| RLS tests (own notes only, no delete) | `convex-helpers/packages/convex-helpers/server/rowLevelSecurity.test.ts` | 41-109 |

**Append-only table via RLS** (not in repo, but trivial):
```ts
myAppendOnlyTable: {
  read: async () => true,
  insert: async () => true,
  modify: async () => false, // prevents patch, replace, delete
}
```

**Elegance note**: RLS is best as a safety net, not primary authorization. The
rules lack context about user intent, making complex scenarios convoluted.
The `rlsExample.ts` file is the cleanest demonstration of the pattern.

---

## Pattern 3: Custom Functions (Auth, Middleware, Restricted ctx)

**What**: Wrap `query`/`mutation`/`action` builders to inject behavior before
every function — auth checks, ctx modification, argument interception.

**Guarantee level**: 2-3 (convention + lint, optionally with runtime db wrapping)

### Core Implementation
- `convex-helpers/packages/convex-helpers/server/customFunctions.ts`
  - `customQuery()` lines 265-291
  - `customMutation()` lines 353-379
  - `customAction()` lines 445-479
  - `customCtx()` lines 173-193

### Examples in the Repo

| Pattern | File | Key lines |
|---------|------|-----------|
| Auth-checked query/mutation with RLS | `convex-helpers/convex/rlsExample.ts` | 39-51 |
| Test-environment-only functions | `convex-helpers/convex/testingFunctions.ts` | 10-47 |
| Session-based custom functions | `sessions/convex/lib/sessions.ts` | 22-50 |
| API key validation | `convex-helpers/packages/convex-helpers/server/customFunctions.test.ts` | 81-108 |
| Trigger-wrapped mutations | `convex-helpers/convex/triggersExample.ts` | 72-76 |

**Most elegant example**: The session wrapper in `sessions/convex/lib/sessions.ts`.
Clean, minimal — consumes a `sessionId` arg, looks up the session, provides it
on `ctx`. The function author never sees the session plumbing.

### Parametrized Custom Functions

From the Authorization article (not in repo as code, but the pattern is):
```ts
export const teamMutation = customMutation(mutation, {
  args: { teamId: v.id("teams") },
  input: async (ctx, args, opts: { role: Role }) => { ... }
});
// Usage forces specifying role:
export const suspendUser = teamMutation({ role: "admin", ... });
```

This is the most scalable auth pattern — type-safe, self-documenting, forces
every endpoint to declare its access level.

---

## Pattern 4: Explicit Wrapper Functions

**What**: Encapsulate all writes to a table in named functions. No triggers,
no magic — just functions you call instead of raw `ctx.db`.

**Guarantee level**: 1 (convention only, unless combined with lint/RLS)

| Pattern | File | Key lines |
|---------|------|-----------|
| User upsert/delete from Clerk webhooks | `users-and-clerk-webhooks/convex/users.ts` | 12-42 |
| User store with auth check | `users-and-clerk/convex/users.ts` | 3-35 |
| CRUD factory (generic for any table) | `convex-helpers/packages/convex-helpers/server/crud.ts` | 56-180 |
| Relationship traversal helpers | `convex-helpers/packages/convex-helpers/server/relationships.ts` | 25-150 |
| Rate-limited operations | `convex-helpers/packages/convex-helpers/server/rateLimit.ts` | 79-104 |
| Migration batch wrapper | `convex-helpers/packages/convex-helpers/server/migrations.ts` | 43-48 |

**Most elegant example**: The CRUD helper in `crud.ts`. For simple tables, you
get `create`/`read`/`update`/`destroy` with schema-validated args in one call.
Combined with RLS, it provides both convenience and safety.

**Append-only table via wrapper** (not in repo, but straightforward):
```ts
// Only export this — no update/delete function exists
async function appendToLog(ctx: MutationCtx, entry: LogEntry) {
  return ctx.db.insert("auditLog", { ...entry, timestamp: Date.now() });
}
```

---

## Pattern 5: Component Isolation (Hardest Guarantee)

**What**: Put tables inside a `defineComponent()`. External code cannot access
them — only the component's exported API exists.

**Guarantee level**: 4 (hard isolation — no bypass possible)

### Components in This Repo

| Component | Location | Internal Tables | API Surface |
|-----------|----------|-----------------|-------------|
| Aggregate | `inlined-components/aggregate/` | `btree`, `btreeNode` | insert, delete, count, sum, at, offset |
| Workflow | `inlined-components/workflow/` | `config`, `workflows`, `steps`, `events`, `onCompleteFailures` | create, getStatus, cancel, cleanup |
| Workpool | `inlined-components/workpool/` | `globals`, `internalState`, `runStatus`, `work`, `pendingStart`, `pendingCompletion`, `pendingCancelation` | enqueue, status, cancel |
| Rate Limiter | `inlined-components/rate-limiter/` | `rateLimits` | limit, check, reset |
| Action Retrier | `inlined-components/action-retrier/` | `runs` | run, status, cancel |
| Action Cache | `inlined-components/action-cache/` | `values`, `metadata` | fetch, remove |
| Presence | `inlined-components/presence/` | `presence`, `sessions`, `roomTokens`, `sessionTokens`, `sessionTimeouts` | heartbeat, list, disconnect |
| Crons | `inlined-components/crons/` | `crons` | register, list, get, delete |
| Migrations | `inlined-components/migrations/` | `migrations` | define, runOne, getStatus, cancel |
| Resend | `inlined-components/resend/` | `content`, `nextBatchRun`, `lastOptions`, `deliveryEvents`, `emails` | sendEmail, status, get |
| WorkOS AuthKit | `inlined-components/workos-authkit/` | `events`, `users` | getAuthUser, events |
| Stripe | `inlined-components/stripe/` | `customers`, `subscriptions`, `checkout_sessions`, `payments`, `invoices` | createCheckoutSession, getCustomer, cancelSubscription |

### Component Structure Pattern

Every component follows the same layout:
```
src/component/
  convex.config.ts    <- defineComponent("name")
  schema.ts           <- private tables
  public.ts / lib.ts  <- exported mutations/queries (the ONLY way in)
src/client/
  index.ts            <- TypeScript wrapper class for ergonomic usage
```

### Component Dependency Graph
```
stripe, aggregate, rate-limiter, action-retrier,
action-cache, presence, crons, migrations     -> standalone

workpool                                       -> standalone
workflow                                       -> workpool
resend                                         -> workpool + rate-limiter
workos-authkit                                 -> workpool
```

**Most elegant examples**:

1. **Aggregate** — the cleanest demonstration of component isolation enforcing
   a data structure invariant. The B-tree must stay balanced; no external code
   can corrupt it because the tables are unreachable. The `.trigger()` API
   bridges the isolation boundary elegantly.

2. **Workpool** — the state machine (7 internal tables!) is complex, but the
   external API is just `enqueue`/`status`/`cancel`. The invariants documented
   in the README (single consumer for pendingStart, exactly-once onComplete)
   are guaranteed by isolation, not convention.

**Append-only table via component**:
Build a component whose public API only has `insert()` and `list()` — no
`patch`/`delete` endpoints. The internal table is unreachable.

---

## Pattern 6: Schema Validators as Type Guardrails

**What**: Define validators once in schema, reuse everywhere. Types flow
end-to-end from schema to client.

**Guarantee level**: 5 for schema enforcement (engine-level), 2 for reuse discipline

### Validator Reuse via Schema Introspection

| Pattern | File | Key lines |
|---------|------|-----------|
| `schema.tables.X.validator.fields` extraction | `inlined-components/workpool/src/component/stats.ts` | 110 |
| Extract field validator for reuse | `inlined-components/crons/src/component/public.ts` | 46 |
| Spread validator fields into return type | `inlined-components/aggregate/src/component/inspect.ts` | 110, 127 |
| `omit()` on schema validators | `inlined-components/resend/src/component/lib.ts` | 345 |

### Validator Manipulation Helpers

| Helper | Location | What it does |
|--------|----------|--------------|
| `partial()` | `convex-helpers/packages/convex-helpers/validators.ts:79-149` | Makes all fields optional |
| `pick()` | `convex-helpers/packages/convex-helpers/index.ts:49` | Select specific fields |
| `omit()` | `convex-helpers/packages/convex-helpers/validators.ts:113` | Remove specific fields |
| `typedV()` | `convex-helpers/packages/convex-helpers/validators.ts:474-514` | Schema-aware `v.id()` and `v.doc()` |
| `doc()` | `convex-helpers/packages/convex-helpers/validators.ts:425-454` | Full document validator with system fields |
| `systemFields()` | `convex-helpers/packages/convex-helpers/validators.ts:284-316` | Just `_id` and `_creationTime` |
| `Table()` | `convex-helpers/packages/convex-helpers/server.ts:33-57` | Bundle validators for schema + functions |

### The `Table()` Helper

Best single-file example of the "define once, use everywhere" pattern:
`convex-helpers/packages/convex-helpers/server/table.test.ts:24-78`

```ts
const Example = Table("table_example", { foo: v.string(), bar: v.number() });
// Example.table        -> for defineSchema
// Example.doc          -> v.object with system fields
// Example.withoutSystemFields -> for insert args
// Example._id          -> v.id("table_example")
```

### Object Validator Methods (.extend, .pick, .omit, .partial)

Documented in `docs/docs/functions/validation.mdx:243-260`:
```ts
const publicUser = userValidator.pick("name", "profileUrl");
const userPatch = userValidator.omit("status").partial();
```

---

## Pattern 7: ESLint Enforcement

**What**: `no-restricted-imports` to prevent importing raw `mutation`/`query`
from `_generated/server`.

**Guarantee level**: 2 (tooling convention — can be disabled)

### In This Repo

- `@convex-dev/eslint-plugin` with recommended config used in:
  - `nextjs-app-router/eslint.config.mjs:6,23`
  - `nextjs-pages-router/eslint.config.mjs:6,23`
  - All 12 `inlined-components/*/eslint.config.js` files
- No explicit `no-restricted-imports` rules found configured in this repo,
  but the pattern is extensively documented in the README and blog posts.

### Recommended Rule (from blog posts)

```js
"no-restricted-imports": ["error", {
  patterns: [{
    group: ["*/_generated/server"],
    importNames: ["query", "mutation", "action"],
    message: "Use functions.ts for query, mutation, or action",
  }],
}],
```

---

## Elegance Rankings (Opinion)

Having surveyed all patterns in the repo, here's what looks most practical:

### For computed/denormalized fields:
1. **Aggregate component + trigger** — strongest guarantee, one-line wiring,
   O(log n) queries. Best for counts, sums, rankings.
   See: `inlined-components/aggregate/example/convex/leaderboard.ts`
2. **Trigger on the table** — good for simple derivations (concat fields,
   compute length). Watch for infinite recursion (check before patching).
   See: `convex-helpers/packages/convex-helpers/server/triggers.test.ts:46-54`
3. **Explicit wrapper function** — simplest, most discoverable, no magic.
   Best when you have few write sites.

### For cascade deletes:
1. **Trigger** — handles it atomically, recursive triggers handle deep graphs.
   See: `convex-helpers/packages/convex-helpers/README.md:1134-1141`
2. **Explicit function** — fine for simple cases, but easy to forget a call site.

### For restricted table interfaces (append-only, read-only, field restrictions):
1. **Component** — unbypassable. If the API doesn't expose delete, delete
   doesn't exist. Period.
2. **RLS with `modify: () => false`** — good safety net within the app boundary.
   See: `convex-helpers/convex/rlsExample.ts`
3. **Custom function replacing ctx.db** — can surgically remove operations.
   Return `ctx: { db: undefined, append: myAppendFn }`.

### For authorization:
1. **Parametrized custom function** (e.g., `teamMutation({ role: "admin" })`)
   — forces every endpoint to declare its access level, type-safe.
2. **Endpoint-specific checks** — most context about user intent.
3. **RLS** — last line of defense, best for compliance auditing.

### The "gold standard" setup:
```
schema.ts       -> validators defined once, exported
functions.ts    -> customMutation wrapping triggers.wrapDB + auth + RLS
.eslintrc       -> no-restricted-imports enforcing functions.ts
components/     -> isolated tables for data structures with internal invariants
```

This gives you layers 2-5 simultaneously.

---

## Quick Reference: "I want to enforce X"

| Constraint | Best pattern | Runner-up | Where to look |
|------------|-------------|-----------|---------------|
| Computed field | Trigger | Explicit wrapper | `triggers.test.ts:46-54` |
| Denormalized count/sum | Aggregate component | Trigger | `aggregate/example/convex/leaderboard.ts` |
| Cascade delete | Trigger | Explicit wrapper | `README.md:1134-1141` |
| Append-only table | Component (no delete API) | RLS `modify: false` | Build your own; `rlsExample.ts` for RLS |
| Immutable field | Trigger (throw on change) | RLS modify rule | `triggersExample.ts` |
| Auth required | Custom function | Endpoint check | `rlsExample.ts:39-51` |
| Role-based access | Parametrized custom fn | Custom function | Authorization blog post |
| Valid email format | Trigger (throw on invalid) | Schema validator | Triggers blog post |
| Foreign key integrity | Trigger / explicit fn | Relationship helpers | `relationships.ts` |
| Rate limiting | Rate Limiter component | `rateLimit.ts` helper | `inlined-components/rate-limiter/` |
| Exactly-once execution | Workpool/Workflow component | Scheduler | `inlined-components/workflow/` |
| No direct table access | Component isolation | RLS + lint | Any component in `inlined-components/` |
