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
internal dependency by other components (e.g. workos-authkit).

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

## How these copies were created

Each component was shallow-cloned and copied in, with `.git`, `.github`, and
lock files removed:

```sh
# Example for stripe (same pattern for all three)
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
