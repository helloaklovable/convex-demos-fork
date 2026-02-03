# Inlined Convex Components

This directory contains full copies of official
[Convex components](https://convex.dev/components) — self-contained backend
modules that bundle functions, schemas, and data in an isolated sandbox. Having
the source here lets Claude Code read component internals, schemas, and usage
examples without fetching from npm or GitHub at runtime.

For background on how components work, see
[`docs/docs/components/`](../docs/docs/components/) (inlined upstream docs) and
each component's own `README.md`.

---

## Component inventory

| Directory | NPM package | Version | Description |
|-----------|-------------|---------|-------------|
| [`stripe/`](#stripe) | `@convex-dev/stripe` | 0.1.3 | Stripe payments, subscriptions, and billing integration |
| [`workos-authkit/`](#workos-authkit) | `@convex-dev/workos-authkit` | 0.1.5 | WorkOS AuthKit user sync, webhooks, and auth actions |
| [`workpool/`](#workpool) | `@convex-dev/workpool` | 0.3.1 | Async work pools with parallelism limits, retries, and completion callbacks |
| [`rate-limiter/`](#rate-limiter) | `@convex-dev/rate-limiter` | 0.3.2 | Application-layer rate limiting with token bucket and fixed window algorithms |
| [`presence/`](#presence) | `@convex-dev/presence` | 0.3.0 | Real-time user presence tracking in rooms with heartbeat sessions |
| [`migrations/`](#migrations) | `@convex-dev/migrations` | 0.3.1 | Stateful, resumable data migrations with batch processing |
| [`aggregate/`](#aggregate) | `@convex-dev/aggregate` | 0.2.1 | O(log n) counts, sums, rankings, and offset pagination via B-tree |
| [`workflow/`](#workflow) | `@convex-dev/workflow` | 0.3.4 | Durable multi-step workflows with events, retries, and nested workflows |
| [`action-retrier/`](#action-retrier) | `@convex-dev/action-retrier` | 0.3.0 | Retry idempotent actions with exponential backoff |
| [`crons/`](#crons) | `@convex-dev/crons` | 0.2.0 | Dynamic runtime cron job registration and management |
| [`action-cache/`](#action-cache) | `@convex-dev/action-cache` | 0.3.0 | Cache expensive action results with TTL and invalidation |
| [`resend/`](#resend) | `@convex-dev/resend` | 0.2.3 | Durable email delivery via Resend with batching, retries, and webhooks |

---

## How components work (quick primer)

A Convex component is a mini backend that lives inside your app:

1. **Install from npm** — `npm i @convex-dev/stripe`
2. **Register in `convex/convex.config.ts`** — `app.use(stripe)`
3. **Run `npx convex dev`** — generates typed API under `components.*`
4. **Call from your functions** — `ctx.runQuery(components.stripe.public.getCustomer, { ... })`

Components have their own database tables, file storage, and scheduler — they
cannot read your app's tables or call your app's functions unless you pass
references in explicitly. See
[`docs/docs/components/using.mdx`](../docs/docs/components/using.mdx) for the
full guide.

---

## Stripe

**Repo:** <https://github.com/get-convex/stripe> · **Commit:** `b46282f`

Integrates Stripe payments, subscriptions, and billing. Provides a
`StripeSubscriptions` client class and automatic webhook-driven data sync.

### Key files

| Path | Purpose |
|------|---------|
| `stripe/README.md` | Full documentation with Quick Start, API reference, and DB schema |
| `stripe/src/client/index.ts` | **`StripeSubscriptions` class** — main client entry point |
| `stripe/src/client/types.ts` | TypeScript types for contexts and event handlers |
| `stripe/src/component/schema.ts` | Component data model (5 tables: customers, subscriptions, payments, invoices, checkout_sessions) |
| `stripe/src/component/convex.config.ts` | Component definition (`defineComponent("stripe")`) |
| `stripe/src/component/public.ts` | Public queries and mutations (getCustomer, listSubscriptions, listPayments, etc.) |
| `stripe/src/component/private.ts` | Internal webhook-processing mutations (subscription, customer, invoice, payment handlers) |

### Example app

| Path | Purpose |
|------|---------|
| `stripe/example/convex/convex.config.ts` | Registers the stripe component via `app.use(stripe)` |
| `stripe/example/convex/stripe.ts` | Full usage: checkout, subscriptions, org billing, customer portal |
| `stripe/example/convex/http.ts` | Webhook route setup with custom event handlers |

### Client API (`StripeSubscriptions`)

```ts
import { StripeSubscriptions } from "@convex-dev/stripe";
import { components } from "./_generated/api";

const stripe = new StripeSubscriptions(components.stripe, {
  stripeSecretKey: process.env.STRIPE_SECRET_KEY!,
});
```

Key methods:
- `createCheckoutSession()` — payment/subscription/setup checkout
- `getOrCreateCustomer()` — idempotent Stripe customer creation
- `createCustomerPortalSession()` — billing portal URL
- `cancelSubscription()` / `reactivateSubscription()`
- `updateSubscriptionQuantity()` — seat-based pricing
- `registerRoutes(http)` — webhook endpoint at `/stripe/webhook`

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `customers` | stripeCustomerId, email, name, metadata | by_stripe_customer_id, by_email |
| `subscriptions` | stripeSubscriptionId, status, priceId, userId?, orgId? | by_stripe_subscription_id, by_org_id, by_user_id |
| `payments` | stripePaymentIntentId, amount, currency, status | by_stripe_payment_intent_id, by_org_id, by_user_id |
| `invoices` | stripeInvoiceId, status, amountDue, amountPaid | by_stripe_invoice_id, by_stripe_subscription_id |
| `checkout_sessions` | stripeCheckoutSessionId, status, mode | by_stripe_checkout_session_id |

---

## WorkOS AuthKit

**Repo:** <https://github.com/get-convex/workos-authkit> · **Commit:** `7ad9252`

Syncs WorkOS AuthKit user data to Convex via webhooks. Supports event-driven
workflows and authentication/registration action handlers.

### Key files

| Path | Purpose |
|------|---------|
| `workos-authkit/README.md` | Full documentation with setup, events, and actions guide |
| `workos-authkit/src/client/index.ts` | **`AuthKit` class** — main client entry point |
| `workos-authkit/src/client/types.ts` | TypeScript context and utility types |
| `workos-authkit/src/component/schema.ts` | Component data model (2 tables: events, users) |
| `workos-authkit/src/component/convex.config.ts` | Component definition (`defineComponent("workOSAuthKit")`) — uses workpool internally |
| `workos-authkit/src/component/lib.ts` | Core logic: webhook event processing, user CRUD, auth user lookup |

### Example app

| Path | Purpose |
|------|---------|
| `workos-authkit/example/convex/convex.config.ts` | Registers the workOSAuthKit component |
| `workos-authkit/example/convex/auth.ts` | Event handlers (user.created/updated/deleted), action handlers (registration/authentication) |
| `workos-authkit/example/convex/http.ts` | Route registration via `authKit.registerRoutes(http)` |
| `workos-authkit/example/convex/schema.ts` | App-side users table synced from WorkOS events |

### Client API (`AuthKit`)

```ts
import { AuthKit } from "@convex-dev/workos-authkit";
import { components } from "./_generated/api";

const authKit = new AuthKit(components.workOSAuthKit);
```

Key methods:
- `getAuthUser(ctx)` — retrieve authenticated user from component's user table
- `events(handlers)` — define handlers for WorkOS user lifecycle events
- `actions(handlers)` — define allow/deny handlers for authentication and registration
- `registerRoutes(http)` — register `/workos/webhook` and `/workos/action` endpoints

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `events` | eventId, event, updatedAt? | by_eventId |
| `users` | id (WorkOS user ID), email, firstName?, lastName?, emailVerified, profilePictureUrl?, lastSignInAt?, metadata | by_id |

### Dependencies

This component uses `@convex-dev/workpool` internally for reliable event queue
processing (also inlined in this directory as [`workpool/`](#workpool)).

---

## Workpool

**Repo:** <https://github.com/get-convex/workpool> · **Commit:** `049d4ce`

Manages pools of async work with configurable parallelism limits, exponential
backoff retries, and completion callbacks. Used both standalone and as an
internal dependency by other components (e.g. workos-authkit, workflow, resend).

### Key files

| Path | Purpose |
|------|---------|
| `workpool/README.md` | Full documentation with configuration, usage, and architecture |
| `workpool/src/client/index.ts` | **`Workpool` class** — main client entry point |
| `workpool/src/client/utils.ts` | Helper types and utilities |
| `workpool/src/component/schema.ts` | Component data model (7 tables: globals, internalState, runStatus, work, pendingStart, pendingCompletion, pendingCancelation) |
| `workpool/src/component/convex.config.ts` | Component definition (`defineComponent("workpool")`) |
| `workpool/src/component/lib.ts` | Public mutations/queries: enqueue, cancel, status |
| `workpool/src/component/loop.ts` | Core orchestration state machine (663 lines) |
| `workpool/src/component/worker.ts` | Job execution wrappers |
| `workpool/src/component/complete.ts` | Completion/retry handler |
| `workpool/src/component/kick.ts` | Main loop scheduling trigger |
| `workpool/src/component/recovery.ts` | Stalled job recovery |
| `workpool/src/component/crons.ts` | Periodic recovery cron (every 30 min) |
| `workpool/src/component/stats.ts` | Event tracking and reporting |
| `workpool/src/component/logging.ts` | Structured logging system |
| `workpool/src/component/config.ts` | Runtime configuration management |
| `workpool/src/component/shared.ts` | Shared types, constants, and time utilities |
| `workpool/src/component/danger.ts` | Data cleanup utilities (clear old jobs) |

### Example app

| Path | Purpose |
|------|---------|
| `workpool/example/convex/convex.config.ts` | Registers 3 pool instances: smallPool, bigPool, serializedPool |
| `workpool/example/convex/example.ts` | Full usage: enqueue mutations/actions/queries, cancel, status, onComplete |
| `workpool/example/convex/schema.ts` | Simple data table for demo |

### Client API (`Workpool`)

```ts
import { Workpool } from "@convex-dev/workpool";
import { components } from "./_generated/api";

const pool = new Workpool(components.emailWorkpool, {
  maxParallelism: 10,
  retryActionsByDefault: true,
  defaultRetryBehavior: {
    maxAttempts: 3,
    initialBackoffMs: 1000,
    base: 2,
  },
});
```

Key methods:
- `enqueueAction(ctx, fn, args, opts?)` / `enqueueActionBatch()`
- `enqueueMutation(ctx, fn, args, opts?)` / `enqueueMutationBatch()`
- `enqueueQuery(ctx, fn, args, opts?)` / `enqueueQueryBatch()`
- `cancel(ctx, workId)` / `cancelAll(ctx)`
- `status(ctx, workId)` / `statusBatch(ctx, workIds)`
- `defineOnComplete(options)` — completion callback helper

Options per enqueue: `retry`, `onComplete`, `context`, `runAt`, `runAfter`.

### Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `globals` | maxParallelism, logLevel | Pool-wide configuration singleton |
| `internalState` | generation, segmentCursors, running[], report | Orchestration state machine |
| `runStatus` | state (running/scheduled/idle), generation | Loop lifecycle tracking |
| `work` | fnType, fnHandle, fnName, fnArgs, attempts, onComplete, retryBehavior | Individual job records |
| `pendingStart` | workId, segment | Jobs queued for execution |
| `pendingCompletion` | workId, segment | Completed jobs awaiting processing |
| `pendingCancelation` | workId, segment | Cancellation requests |

### Architecture

The workpool uses a segment-based time-bucketing system (100 ms slices) to
reduce database conflicts. A single main loop (`loop.ts`) acts as the heartbeat,
processing pending starts, completions, and cancellations. A generation counter
ensures only one loop instance runs at a time. Recovery runs via a 30-minute
cron job to handle stalled work.

---

## Rate Limiter

**Repo:** <https://github.com/get-convex/rate-limiter> · **Commit:** `df9b3d8`

Application-layer rate limiting with type-safe named limits. Supports token
bucket and fixed window algorithms with configurable sharding for high
throughput.

### Key files

| Path | Purpose |
|------|---------|
| `rate-limiter/README.md` | Full documentation with algorithms, sharding, and React hook guide |
| `rate-limiter/src/client/index.ts` | **`RateLimiter` class** — main client entry point |
| `rate-limiter/src/component/schema.ts` | Component data model (1 table: rateLimits) |
| `rate-limiter/src/component/convex.config.ts` | Component definition (`defineComponent("rateLimiter")`) |
| `rate-limiter/src/component/lib.ts` | Core mutations/queries: rateLimit, checkRateLimit, getValue, resetRateLimit, clearAll |
| `rate-limiter/src/component/internal.ts` | Internal implementation: checkRateLimitOrThrow, getShard, configWithDefaults |
| `rate-limiter/src/react/index.ts` | **`useRateLimit` hook** — client-side rate limit checking |

### Example app

| Path | Purpose |
|------|---------|
| `rate-limiter/example/convex/convex.config.ts` | Registers the rateLimiter component |
| `rate-limiter/example/convex/example.ts` | Usage with named limits, per-key limits, React hooks |
| `rate-limiter/example/convex/playground.ts` | Interactive rate limit testing |

### Client API (`RateLimiter`)

```ts
import { RateLimiter, MINUTE, HOUR } from "@convex-dev/rate-limiter";
import { components } from "./_generated/api";

const limiter = new RateLimiter(components.rateLimiter, {
  sendMessage: { kind: "token bucket", rate: 10, period: MINUTE },
  createAccount: { kind: "fixed window", rate: 5, period: HOUR },
});
```

Key methods:
- `limit(ctx, name, opts?)` — consume tokens, returns `{ ok, retryAfter }`
- `check(ctx, name, opts?)` — check without consuming
- `reset(ctx, name, opts?)` — reset a rate limit
- `getValue(ctx, name, opts?)` — get current value, timestamp, shard count, and config

Options per call: `key` (per-user), `count`, `reserve`, `throws`, inline `config`.

Time constants: `SECOND`, `MINUTE`, `HOUR`, `DAY`, `WEEK`.

React hook: `useRateLimit(getRateLimitValueQuery)` — returns `{ status: { ok, retryAt }, check }`.

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `rateLimits` | name, key?, shard, value, ts | name: [name, key, shard] |

### Algorithms

- **Token bucket** — smooth rate over time with configurable burst capacity
- **Fixed window** — hard cap per time window with optional UTC start alignment
- Both support **sharding** for high-throughput scenarios (power-of-two selection)

---

## Presence

**Repo:** <https://github.com/get-convex/presence> · **Commit:** `a7502af`

Real-time user presence tracking in "rooms" (chat rooms, documents, games).
Heartbeat-based sessions with graceful disconnect handling. No polling required.

### Key files

| Path | Purpose |
|------|---------|
| `presence/README.md` | Full documentation with setup and React hook guide |
| `presence/src/client/index.ts` | **`Presence` class** — main client entry point |
| `presence/src/component/schema.ts` | Component data model (5 tables: presence, sessions, roomTokens, sessionTokens, sessionTimeouts) |
| `presence/src/component/convex.config.ts` | Component definition (`defineComponent("presence")`) |
| `presence/src/component/public.ts` | All public API functions: heartbeat, list, disconnect, updateRoomUser, etc. |
| `presence/src/react/index.ts` | **`usePresence` hook** — automatic heartbeats and presence listing |

### Example app

| Path | Purpose |
|------|---------|
| `presence/example/convex/convex.config.ts` | Registers the presence component |
| `presence/example/convex/presence.ts` | Wrapper mutations/queries: heartbeat, list, disconnect, updateRoomUser |

### Client API (`Presence`)

```ts
import { Presence } from "@convex-dev/presence";
import { components } from "./_generated/api";

const presence = new Presence(components.presence);
```

Key methods:
- `heartbeat(ctx, roomId, userId, sessionId, interval)` — keepalive, returns `{ roomToken, sessionToken }`
- `list(ctx, roomToken, limit?)` — list users in room (by token for caching)
- `disconnect(ctx, sessionToken)` — graceful disconnect
- `updateRoomUser(ctx, roomId, userId, data?)` — update custom presence data
- `listRoom(ctx, roomId, onlineOnly?, limit?)` — list room users by roomId
- `listUser(ctx, userId, onlineOnly?, limit?)` — list all rooms a user is in
- `removeRoomUser(ctx, roomId, userId)` — admin remove user
- `removeRoom(ctx, roomId)` — admin remove entire room

React hook: `usePresence(heartbeatMutation, listQuery, roomId, userId, opts?)` — handles heartbeats automatically.

Also exports: `FacePile` component and CSS for rendering present users.

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `presence` | roomId, userId, online, lastDisconnected, data? | user_online_room, room_order |
| `sessions` | roomId, userId, sessionId | room_user_session, sessionId |
| `roomTokens` | token, roomId | token, room |
| `sessionTokens` | token, sessionId | token, sessionId |
| `sessionTimeouts` | sessionId, scheduledFunctionId | sessionId |

---

## Migrations

**Repo:** <https://github.com/get-convex/migrations> · **Commit:** `4337f9b`

Stateful, resumable data migrations with batch processing. Track progress,
resume from failures, and run from CLI or server functions.

### Key files

| Path | Purpose |
|------|---------|
| `migrations/README.md` | Full documentation with CLI usage and workflow guide |
| `migrations/src/client/index.ts` | **`Migrations` class** — main client entry point |
| `migrations/src/client/log.ts` | Status logging and CLI output formatting |
| `migrations/src/component/schema.ts` | Component data model (1 table: migrations) |
| `migrations/src/component/convex.config.ts` | Component definition (`defineComponent("migrations")`) |
| `migrations/src/component/lib.ts` | Backend mutations/queries: migrate, getStatus, cancel, cancelAll, clearAll |

### Example app

| Path | Purpose |
|------|---------|
| `migrations/example/convex/convex.config.ts` | Registers the migrations component |
| `migrations/example/convex/example.ts` | Defines migrations: setDefaultValue, clearField, validateRequiredField, convertUnionField, runners, postDeploy |

### Client API (`Migrations`)

```ts
import { Migrations } from "@convex-dev/migrations";
import { components } from "./_generated/api";

const migrations = new Migrations(components.migrations, {
  internalMutation: internalMutation,
  defaultBatchSize: 100,
});
```

Key methods:
- `define({ table, migrateOne, batchSize?, customRange?, parallelize? })` — define a migration over a table
- `runner(specificMigrationOrSeries?)` — create a CLI-runnable migration runner mutation
- `runOne(ctx, fnRef, opts?)` — start a single migration programmatically
- `runSerially(ctx, fnRefs)` — run migrations in sequence
- `getStatus(ctx, config?)` — query migration progress
- `cancel(ctx, migration)` / `cancelAll(ctx)` — cancel migrations

Also exports `runToCompletion(ctx, component, fnRef)` for running synchronously in tests/actions.

CLI usage: `npx convex run migrations:run '{"fn": "migrations:myMigration"}'`

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `migrations` | name, cursor, isDone, workerId?, error?, processed, latestStart, latestEnd? | name, isDone |

---

## Aggregate

**Repo:** <https://github.com/get-convex/aggregate> · **Commit:** `0f9dfd3`

O(log n) counts, sums, rankings, and offset-based pagination over sorted data.
Uses an internal B-tree for efficient aggregation with full transactional and
reactive guarantees.

### Key files

| Path | Purpose |
|------|---------|
| `aggregate/README.md` | Full documentation with leaderboard, pagination, and grouping examples |
| `aggregate/src/client/index.ts` | **`Aggregate`, `DirectAggregate`, `TableAggregate` classes** — main entry points |
| `aggregate/src/client/positions.ts` | Internal position/bound utilities |
| `aggregate/src/component/schema.ts` | Component data model (2 tables: btree, btreeNode) |
| `aggregate/src/component/convex.config.ts` | Component definition (`defineComponent("aggregate")`) |
| `aggregate/src/component/btree.ts` | Core B-tree implementation and aggregation logic (33 KB) |
| `aggregate/src/component/public.ts` | Public mutation handlers for insert/delete/replace |
| `aggregate/src/component/compare.ts` | Key comparison utilities |
| `aggregate/src/component/inspect.ts` | Debugging/inspection utilities |

### Example app

| Path | Purpose |
|------|---------|
| `aggregate/example/convex/convex.config.ts` | Registers the aggregate component |
| `aggregate/example/convex/leaderboard.ts` | Game leaderboard with counts/sums/rankings |
| `aggregate/example/convex/photos.ts` | Offset-based pagination with namespaces |
| `aggregate/example/convex/stats.ts` | DirectAggregate usage without tables |
| `aggregate/example/convex/shuffle.ts` | Random access and shuffling |

### Client API

Three classes with increasing abstraction:

**`DirectAggregate`** — manual key/id management:
```ts
import { DirectAggregate } from "@convex-dev/aggregate";
const agg = new DirectAggregate<{ Key: number; Id: string }>(components.aggregate);
await agg.insert(ctx, { key: score, id: oduserId, sumValue: points });
```

**`TableAggregate`** — automatic sync from Convex tables:
```ts
import { TableAggregate } from "@convex-dev/aggregate";
const leaderboard = new TableAggregate<{
  Key: number; DataModel: DataModel; TableName: "scores";
}>(components.aggregate, {
  sortKey: (doc) => doc.score,
  sumValue: (doc) => doc.points,
});
```

Shared query methods (all classes):
- `count(ctx, opts?)` / `countBatch()` — count items between bounds
- `sum(ctx, opts?)` / `sumBatch()` — sum values between bounds
- `at(ctx, offset, opts?)` / `atBatch()` — get item at index
- `indexOf(ctx, key, opts?)` — get rank of key
- `min(ctx, opts?)` / `max(ctx, opts?)` — get extreme items
- `random(ctx, opts?)` — get random item
- `paginate(ctx, opts?)` — cursor-based pagination
- `iter(ctx, opts?)` — async iteration

TableAggregate extras:
- `trigger()` / `idempotentTrigger()` — automatic sync via database triggers
- `indexOfDoc(ctx, doc)` — rank from document

### Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `btree` | root, namespace?, maxNodeSize | One per namespace — B-tree root reference |
| `btreeNode` | items[{k, v, s}], subtrees[], aggregate?{count, sum} | Internal B-tree nodes |

---

## Workflow

**Repo:** <https://github.com/get-convex/workflow> · **Commit:** `7a83ef1`

Durable multi-step workflows that survive server restarts and can run for months.
Supports parallel and sequential steps, nested workflows, external events
(human-in-the-loop), and completion callbacks.

### Key files

| Path | Purpose |
|------|---------|
| `workflow/README.md` | Full documentation with step types, events, and limitations |
| `workflow/src/client/index.ts` | **`WorkflowManager` class** — main client entry point |
| `workflow/src/client/workflowContext.ts` | `WorkflowCtx` type with step execution methods |
| `workflow/src/client/step.ts` | `StepExecutor` — step execution engine |
| `workflow/src/client/environment.ts` | Deterministic sandbox for workflow bodies |
| `workflow/src/component/schema.ts` | Component data model (5 tables: workflows, steps, events, config, onCompleteFailures) |
| `workflow/src/component/convex.config.ts` | Component definition — uses workpool internally |
| `workflow/src/component/workflow.ts` | Core workflow queries/mutations: create, getStatus, list, cancel, complete, cleanup |
| `workflow/src/component/journal.ts` | Step journal management |
| `workflow/src/component/event.ts` | Event handling |
| `workflow/src/component/pool.ts` | Workpool integration |

### Example app

| Path | Purpose |
|------|---------|
| `workflow/example/convex/convex.config.ts` | Registers the workflow component |
| `workflow/example/convex/example.ts` | Basic workflow: geocoding + weather API calls with parallel steps |
| `workflow/example/convex/userConfirmation.ts` | Human-in-the-loop with `awaitEvent` for approvals |
| `workflow/example/convex/nestedWorkflow.ts` | Parent-child workflow composition |
| `workflow/example/convex/passingSignals.ts` | Dynamic event creation and ID passing |

### Client API (`WorkflowManager`)

```ts
import { WorkflowManager } from "@convex-dev/workflow";
import { components } from "./_generated/api";

const workflow = new WorkflowManager(components.workflow);
```

Define workflows:
```ts
export const myWorkflow = workflow.define({
  args: { city: v.string() },
  handler: async (ctx, args) => {
    const coords = await ctx.runAction(internal.geo.geocode, { city: args.city });
    const weather = await ctx.runAction(internal.weather.fetch, coords);
    await ctx.runMutation(internal.db.saveResult, { city: args.city, weather });
    return weather;
  },
});
```

Key methods:
- `define({ args, handler, returns? })` — define a durable workflow
- `start(ctx, workflowRef, args, opts?)` — kick off a workflow, returns `WorkflowId`
- `status(ctx, workflowId)` — get status: inProgress / completed / canceled / failed
- `cancel(ctx, workflowId)` — cancel a running workflow
- `list(ctx, opts?)` / `listByName(ctx, name, opts?)` — paginated workflow listing
- `listSteps(ctx, workflowId, opts?)` — list steps in a workflow
- `cleanup(ctx, workflowId)` — clean up completed workflow storage
- `sendEvent(ctx, args)` / `createEvent(ctx, args)` — external event handling

`WorkflowCtx` step methods: `runQuery`, `runMutation`, `runAction`, `runWorkflow`, `awaitEvent`.

Also exports `defineEvent({ name, validator })` for typed event specs.

### Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| `workflows` | name?, workflowHandle, args, onComplete?, runResult?, generationNumber | Workflow instances |
| `steps` | workflowId, stepNumber, step{name, inProgress, fnType, handle, runResult, ...} | Step journal entries |
| `events` | workflowId, name, state{kind: created/sent/waiting/consumed} | External event lifecycle |
| `config` | logLevel?, maxParallelism? | Runtime configuration singleton |
| `onCompleteFailures` | workflowId, result, context, error | Failed completion callback records |

### Dependencies

This component uses `@convex-dev/workpool` internally for step scheduling
(also inlined in this directory as [`workpool/`](#workpool)).

---

## Action Retrier

**Repo:** <https://github.com/get-convex/action-retrier> · **Commit:** `d76fa1e`

Retries idempotent actions with exponential backoff and jitter. Supports
scheduled execution, completion callbacks, and automatic cleanup.

### Key files

| Path | Purpose |
|------|---------|
| `action-retrier/README.md` | Full documentation with backoff configuration and examples |
| `action-retrier/src/client/index.ts` | **`ActionRetrier` class** — main client entry point |
| `action-retrier/src/component/schema.ts` | Component data model (1 table: runs) |
| `action-retrier/src/component/convex.config.ts` | Component definition (`defineComponent("actionRetrier")`) |
| `action-retrier/src/component/public.ts` | Public API: start, status, cancel, cleanup |
| `action-retrier/src/component/run.ts` | Internal run orchestration, heartbeat, retry logic |
| `action-retrier/src/component/crons.ts` | Daily cleanup cron for expired runs |

### Example app

| Path | Purpose |
|------|---------|
| `action-retrier/example/convex/convex.config.ts` | Registers the actionRetrier component |
| `action-retrier/example/convex/example.ts` | Usage: run with retries, runAfter, runAt, onComplete callback, status checking |

### Client API (`ActionRetrier`)

```ts
import { ActionRetrier } from "@convex-dev/action-retrier";
import { components } from "./_generated/api";

const retrier = new ActionRetrier(components.actionRetrier, {
  initialBackoffMs: 250,
  base: 2,
  maxFailures: 4,
});
```

Key methods:
- `run(ctx, actionRef, args?, opts?)` — run action with retries, returns `RunId`
- `runAt(ctx, timestampMs, actionRef, args?, opts?)` — run no earlier than timestamp
- `runAfter(ctx, delayMs, actionRef, args?, opts?)` — run after delay
- `status(ctx, runId)` — returns `{ type: "inProgress" }` or `{ type: "completed", result }`
- `cancel(ctx, runId)` — best-effort cancellation
- `cleanup(ctx, runId)` — manually clean up completed run (auto-cleanup after 7 days)

RunResult: `{ type: "success", returnValue }` | `{ type: "failed", error }` | `{ type: "canceled" }`.

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `runs` | functionHandle, functionArgs, options{initialBackoffMs, base, maxFailures}, state{type, startTime?, completedAt?, result?}, numFailures | by_state: [state.type, state.completedAt] |

---

## Crons

**Repo:** <https://github.com/get-convex/crons> · **Commit:** `967fd48`

Dynamic runtime cron job registration — unlike built-in Convex crons, these can
be created, listed, and deleted at runtime from mutations.

### Key files

| Path | Purpose |
|------|---------|
| `crons/README.md` | Full documentation with cron spec and interval examples |
| `crons/src/client/index.ts` | **`Crons` class** — main client entry point |
| `crons/src/component/schema.ts` | Component data model (1 table: crons) |
| `crons/src/component/convex.config.ts` | Component definition (`defineComponent("crons")`) |
| `crons/src/component/public.ts` | Implementation: register, list, get, del, rescheduler |

### Example app

| Path | Purpose |
|------|---------|
| `crons/example/convex/convex.config.ts` | Registers the crons component |
| `crons/example/convex/example.ts` | Usage: registerDailyCron, self-deleting cron, list/get/delete operations |

### Client API (`Crons`)

```ts
import { Crons } from "@convex-dev/crons";
import { components } from "./_generated/api";

const crons = new Crons(components.crons);
```

Key methods:
- `register(ctx, schedule, func, args, name?)` — schedule a mutation or action; returns cron id
- `list(ctx)` — list all cron jobs
- `get(ctx, { id } | { name })` — get a cron by id or name
- `delete(ctx, { id } | { name })` — delete and deschedule a cron

Schedule types:
- `{ kind: "cron", cronspec: "0 0 * * *", tz?: "America/New_York" }` — cron expression
- `{ kind: "interval", ms: 60000 }` — fixed interval (min 1000 ms)

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `crons` | name?, functionHandle, args, schedule{kind, cronspec/ms, tz?}, schedulerJobId?, executionJobId? | name |

---

## Action Cache

**Repo:** <https://github.com/get-convex/action-cache> · **Commit:** `f474ab1`

Caches expensive action results with optional TTL and invalidation. Automatic
daily cleanup of expired entries.

### Key files

| Path | Purpose |
|------|---------|
| `action-cache/README.md` | Full documentation with TTL, versioning, and forced-update examples |
| `action-cache/src/client/index.ts` | **`ActionCache` class** — main client entry point |
| `action-cache/src/component/schema.ts` | Component data model (2 tables: values, metadata) |
| `action-cache/src/component/convex.config.ts` | Component definition (`defineComponent("actionCache")`) |
| `action-cache/src/component/lib.ts` | Internal API: get, put, remove, removeAll |
| `action-cache/src/component/cache.ts` | Cache lookup and deletion utilities |
| `action-cache/src/component/crons.ts` | Daily purge cron for expired entries |

### Example app

| Path | Purpose |
|------|---------|
| `action-cache/example/convex/convex.config.ts` | Registers the actionCache component |
| `action-cache/example/convex/example.ts` | Two caches: geocoding (7-day TTL) and weather (5-minute TTL) |

### Client API (`ActionCache`)

```ts
import { ActionCache } from "@convex-dev/action-cache";
import { components } from "./_generated/api";

const geocodeCache = new ActionCache(components.actionCache, {
  action: internal.geo.geocode,
  name: "geocode-v1",
  ttl: 7 * 24 * 60 * 60 * 1000, // 7 days
});
```

Key methods:
- `fetch(ctx, args, opts?)` — get cached value or call action on miss; `{ ttl?, force? }`
- `remove(ctx, args)` — remove single cache entry
- `removeAllForName(ctx, opts?)` — clear all entries for this cache name
- `removeAll(ctx, before?)` — clear entire cache (deprecated, use module-level `removeAll`)

Module-level: `removeAll(ctx, component, before?)` — clear all cache entries globally.

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `values` | name, args, value, metadataId? | key: [name, args] |
| `metadata` | valueId, expiresAt | expiresAt |

---

## Resend

**Repo:** <https://github.com/get-convex/resend> · **Commit:** `d347113`

Durable email delivery via Resend with automatic batching, rate limiting,
retries, and webhook-driven status tracking.

### Key files

| Path | Purpose |
|------|---------|
| `resend/README.md` | Full documentation with setup, templates, and webhook guide |
| `resend/src/client/index.ts` | **`Resend` class** — main client entry point |
| `resend/src/component/schema.ts` | Component data model (5 tables: emails, deliveryEvents, content, lastOptions, nextBatchRun) |
| `resend/src/component/convex.config.ts` | Component definition — uses rateLimiter and workpool internally |
| `resend/src/component/shared.ts` | Shared types: Status, EmailEvent, RuntimeConfig |
| `resend/src/component/lib.ts` | Core email queuing, batching, and webhook processing |

### Example app

| Path | Purpose |
|------|---------|
| `resend/example/convex/convex.config.ts` | Registers the resend component |
| `resend/example/convex/example.ts` | Usage: sendOne, testBatch (25 emails), sendWithTemplate, sendManualEmail, handleEmailEvent |
| `resend/example/convex/http.ts` | Webhook handler endpoint |

### Client API (`Resend`)

```ts
import { Resend } from "@convex-dev/resend";
import { components } from "./_generated/api";

const resend = new Resend(components.resend, {
  onEmailEvent: internal.email.handleEmailEvent,
  testMode: false,
});
```

Key methods:
- `sendEmail(ctx, { from, to, subject, html?, text?, template?, ... })` — enqueue email, returns `EmailId`
- `sendEmailManually(ctx, opts, callback)` — send without batching (for attachments etc.)
- `status(ctx, emailId)` — get delivery status
- `get(ctx, emailId)` — full email details
- `cancelEmail(ctx, emailId)` — cancel if not yet sent
- `handleResendEventWebhook(ctx, req)` — process Resend webhook events
- `defineOnEmailEvent(handler)` — helper to define email event callback

Email statuses: waiting, queued, cancelled, sent, delivered, delivery_delayed, bounced, failed.

### Data model

| Table | Key fields | Indexes |
|-------|-----------|---------|
| `emails` | from, to, subject, html?, text?, template?, status, resendId?, segment, finalizedAt | by_status_segment, by_resendId, by_finalizedAt |
| `deliveryEvents` | emailId, resendId, eventType, createdAt, message? | by_emailId_eventType |
| `content` | content (bytes), mimeType, filename?, path? | — |
| `lastOptions` | options (RuntimeConfig snapshot) | — |
| `nextBatchRun` | runId (scheduled function ref) | — |

### Dependencies

This component uses `@convex-dev/rate-limiter` and `@convex-dev/workpool`
internally (both also inlined in this directory).

---

## How these copies were created

Each component was shallow-cloned and copied in, with `.git`, `.github`, and
lock files removed:

```sh
# Example for stripe (same pattern for all components)
git clone --depth 1 https://github.com/get-convex/stripe.git /tmp/convex-stripe
cp -r /tmp/convex-stripe inlined-components/stripe
rm -rf inlined-components/stripe/.git inlined-components/stripe/.github \
       inlined-components/stripe/package-lock.json
rm -rf /tmp/convex-stripe
```

## Syncing all components to latest

Run the script below from the repo root to refresh every component at once:

```sh
#!/usr/bin/env bash
set -euo pipefail

DEST="inlined-components"

declare -A COMPONENTS=(
  [stripe]="https://github.com/get-convex/stripe.git"
  [workos-authkit]="https://github.com/get-convex/workos-authkit.git"
  [workpool]="https://github.com/get-convex/workpool.git"
  [rate-limiter]="https://github.com/get-convex/rate-limiter.git"
  [presence]="https://github.com/get-convex/presence.git"
  [migrations]="https://github.com/get-convex/migrations.git"
  [aggregate]="https://github.com/get-convex/aggregate.git"
  [workflow]="https://github.com/get-convex/workflow.git"
  [action-retrier]="https://github.com/get-convex/action-retrier.git"
  [crons]="https://github.com/get-convex/crons.git"
  [action-cache]="https://github.com/get-convex/action-cache.git"
  [resend]="https://github.com/get-convex/resend.git"
)

for name in "${!COMPONENTS[@]}"; do
  url="${COMPONENTS[$name]}"
  echo "==> Syncing $name from $url"
  rm -rf "$DEST/$name"
  git clone --depth 1 "$url" "/tmp/convex-$name"
  cp -r "/tmp/convex-$name" "$DEST/$name"
  rm -rf "$DEST/$name/.git" "$DEST/$name/.github" "$DEST/$name/package-lock.json"
  rm -rf "/tmp/convex-$name"
done

echo "Done. All components updated."
```

To sync a single component, run just the commands for that entry.

## Adding a new component

1. Add a `git clone --depth 1` + `cp -r` + cleanup block (see pattern above)
2. Add an entry to the [Component inventory](#component-inventory) table
3. Add a detailed section with key files, example app, client API, and data model
4. Update the sync script's `COMPONENTS` array
5. Update the root `README.md` reference
