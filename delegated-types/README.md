# Delegated Types: The Recordables Pattern

A Convex demo implementing the **delegated type** pattern from Ruby on Rails, as
pioneered by Basecamp for modeling content across Basecamp and HEY.

The demo models a simplified content management system where all content shares
uniform metadata and operations while each type retains its own specific
attributes. The killer feature: **immutable recordables with version history** —
editing creates a new version, the old version is never touched, and the full
change log falls out naturally.

---

## Part 1: Convex Primer

Convex is a reactive backend platform. You define your database schema and
server functions in TypeScript. Convex handles persistence, real-time
subscriptions, and type safety from database to UI. A few concepts are
essential before we get into the pattern.

### Tables, Documents, and Schemas

Convex stores **documents** (JSON-like objects) in **tables**. You define the
shape of each table in `convex/schema.ts` using validators from `convex/values`:

```typescript
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  messages: defineTable({
    author: v.string(),
    body: v.string(),
  }),
});
```

Every document automatically gets an `_id` (a globally unique, typed
identifier) and a `_creationTime` (numeric timestamp). You never define these —
they exist on every document.

### References Between Tables

A document can reference another document using `v.id("tableName")`. This is
Convex's typed foreign key:

```typescript
books: defineTable({
  title: v.string(),
  authorId: v.id("authors"),  // typed reference to the authors table
})
```

When you call `ctx.db.get(someId)`, Convex returns the document from the
correct table — the table is encoded in the ID itself.

### Unions and Literal Types

Convex supports union validators. A field can hold one of several types:

```typescript
status: v.union(v.literal("active"), v.literal("archived"), v.literal("trashed"))
```

You can also union across `v.id()` types. A single field can reference
documents from different tables:

```typescript
targetId: v.union(v.id("messages"), v.id("comments"), v.id("documents"))
```

### Indexes

Indexes are declared on tables and enable efficient filtered queries:

```typescript
messages: defineTable({
  channel: v.id("channels"),
  body: v.string(),
}).index("by_channel", ["channel"])
```

Queries use indexes explicitly:

```typescript
const msgs = await ctx.db
  .query("messages")
  .withIndex("by_channel", (q) => q.eq("channel", channelId))
  .collect();
```

### Queries and Mutations

Server-side logic lives in **query** functions (read-only) and **mutation**
functions (read-write). Both validate their arguments at runtime and infer
TypeScript types from validators:

```typescript
export const send = mutation({
  args: { body: v.string(), author: v.string() },
  handler: async (ctx, { body, author }) => {
    await ctx.db.insert("messages", { body, author });
  },
});
```

Mutations are **transactional** — all reads and writes within a single mutation
either fully succeed or fully roll back. There are no partial failures.

### Reactivity

On the frontend, `useQuery(api.messages.list)` subscribes to the query. When
underlying data changes, the component re-renders automatically. There is no
manual cache invalidation — the system handles it.

---

## Part 2: The Pattern in Rails

### The Problem

Applications often have many content types — messages, comments, documents,
uploads — that share metadata (timestamps, ownership, access control) but
differ in their specific attributes. A message has a subject and body; a
comment has just content; an event has start/end times.

How do you model this without repeating yourself?

### Why Not Single Table Inheritance?

Put everything in one table. Add a `type` column to select the class. Every
column any subtype needs must exist on that table:

```
entries
├── id
├── type          ("Message", "Comment", "Event")
├── subject       (messages only — NULL for others)
├── body          (messages and comments)
├── starts_at     (events only — NULL for others)
├── ends_at       (events only — NULL for others)
└── ...
```

The table grows wider with every new type. Most columns are NULL for most rows.
Adding a type means altering this single, ever-growing table.

### Why Not Polymorphic Associations?

Give each type its own table. Shared concerns use a polymorphic foreign key —
`commentable_id` + `commentable_type` — so a comment can belong to any record:

```
comments
├── commentable_type   ("Message", "Document")
├── commentable_id
├── content
└── ...
```

Now you're working from the bottom up. Querying "all content" across types
requires querying each table and merging results. Pagination across types is
painful. Uniform operations (trash, copy, export) must be implemented per-type.

### The Delegated Type Pattern

Invert the relationship. A single **superclass table** points down to "any kind
of content":

```
recordings (superclass — shared metadata only)
├── id
├── recordable_type    ("Message", "Comment", "Document")
├── recordable_id      (FK to the specific type's table)
├── creator_id
├── created_at
└── updated_at

messages (recordable — type-specific)    comments       documents
├── id                                   ├── id          ├── id
├── subject                              └── content     ├── title
└── body                                                 └── body
```

The `recordings` table holds shared metadata plus a type/id pair. Each concrete
type has its own table with only its specific attributes.

In Rails, three parts wire this up:

```ruby
# 1. Superclass declares delegated type
class Recording < ApplicationRecord
  delegated_type :recordable, types: %w[ Message Comment Document ]
end

# 2. Shared module defines reverse association
module Recordable
  extend ActiveSupport::Concern
  included do
    has_one :recording, as: :recordable, touch: true
  end
end

# 3. Each type includes the module
class Message < ApplicationRecord
  include Recordable
end
```

This generates scopes (`Recording.messages`), type checks
(`recording.message?`), accessors (`recording.message`), and more. You query
the single `recordings` table to get any mix of types. You write one controller
for trash/copy/export that works on any recording. Adding a new type means
creating a new table and model — the `recordings` table is never altered.

### The Full Basecamp Extension

Basecamp layers additional patterns on top of the Rails primitive:

- **Buckets**: containers (projects, templates) that hold recordings and
  control access. Recordables know nothing about access — that's the bucket's
  job.
- **Tree structure**: recordings form parent-child hierarchies. A message
  board's children are messages; a message's children are comments. Navigation
  always goes through recordings.
- **Immutable recordables**: recordables are never modified in place. Editing
  creates a new recordable and updates the recording's pointer. The old version
  still exists in the database, untouched.
- **Event history**: an events table logs which recordable a recording pointed
  to at each moment, enabling version history and change logs. As Jeff Hardy
  explains: "we can look at the history of a recording and see all of its
  changes and look at that recordable that is immutable at any moment in time
  to see how it looked."
- **Cheap copies**: copying a recording means creating a new recording row that
  points to the same recordable. No content is duplicated. If a message gets
  copied 100 times, there is still only one message recordable.
- **Capabilities**: each type declares what it supports (commentable,
  subscribable, exportable) via boolean methods. Generic controllers check
  these before acting.

This demo implements the core delegated type pattern plus immutable recordables,
event history, and cheap copies — the combination that Jeffrey Hardy calls the
reason "you can build entirely new features that should take months in like a
week, two weeks."

---

## Part 3: From Rails to Convex

The delegated type pattern maps naturally to Convex. The core data structure —
a lightweight superclass table referencing separate type-specific tables — is
directly expressible with Convex's schema, typed IDs, and unions. The
differences are in how the two platforms express behavior and relationships.

### Structural Mapping

| Rails Concept | Convex Equivalent |
|---|---|
| `recordings` SQL table | `recordings` Convex table |
| `recordable_type` column (string) | `recordableType` field with `v.union(v.literal("message"), ...)` |
| `recordable_id` column (integer FK) | `recordableId` field with `v.union(v.id("messages"), ...)` |
| `messages` SQL table | `messages` Convex table |
| `belongs_to :bucket` | `projectId: v.id("projects")` |
| `creator_id` FK | `creatorId: v.id("users")` |
| `events` SQL table | `events` Convex table |
| `created_at` / `updated_at` | `_creationTime` (automatic) |

### Behavior Mapping

| Rails Concept | Convex Equivalent |
|---|---|
| `delegated_type :recordable, types: [...]` | Schema definition with `v.union()` of `v.literal()` types |
| `Recording.messages` (scope) | Indexed query: `.withIndex("by_project_and_type", q => q.eq(...).eq("recordableType", "message"))` |
| `recording.message?` (type check) | `recording.recordableType === "message"` (TypeScript narrowing) |
| `recording.recordable` (fetch delegate) | `ctx.db.get(recording.recordableId)` (ID encodes the table) |
| `recording.message` (typed fetch) | Helper function with type guard + `ctx.db.get()` |
| Immutable recordables (create new, update pointer) | `insertRecordable` helper that only inserts, never patches |
| Event logging (after_commit callbacks) | Explicit event insert inside the mutation — no hidden callbacks |
| `Recording::Copier` (copy without duplicating content) | `copy` mutation: new recording row, same `recordableId` |
| ActiveRecord callbacks | Logic in mutation handlers (explicit, not implicit) |
| Russian doll caching | Convex reactive queries (automatic, no manual invalidation) |

### What Changes, What Stays the Same

**Stays the same:**
- The core data architecture — one metadata table, separate content tables,
  type/id reference pair.
- The principle that recordings are the unit of organization and querying.
- The principle that recordables are dumb content with no external references.
- Immutability of recordables — editing means creating, never modifying.
- Event history as the mechanism for version tracking.
- The benefit of uniform operations across types.

**Changes:**
- **No ORM magic.** Rails' `delegated_type` generates methods, scopes, and
  associations through metaprogramming. In Convex, you express the same
  relationships explicitly through schema validators and helper functions.
  This is more verbose but completely transparent — there's no hidden behavior.
- **TypeScript replaces Ruby conventions.** Rails relies on naming conventions
  (`recordable_type` maps to class names). Convex uses TypeScript's type system
  — `v.literal("message")` is checked at compile time and validated at runtime.
- **Transactions are built in.** In Rails, creating a recording and its
  recordable together requires an explicit `transaction` block (see Basecamp's
  `Bucket::Recorder#record`). In Convex, every mutation is automatically
  atomic — insert the recordable, insert the recording, log the event, and all
  three either succeed or none do.
- **Reactivity replaces caching.** Rails uses cache keys and Russian doll
  caching to avoid re-rendering. Convex queries are reactive subscriptions —
  when data changes, subscribed components re-render with fresh data. There is
  no cache to invalidate.
- **Append-only is structural, not conventional.** In Rails, the immutability
  of recordables is a team convention enforced by code review. In Convex, we
  enforce it by design: a single `insertRecordable` helper is the only write
  path to recordable tables, and it only inserts. There are no exported
  functions that patch or delete recordable content.

---

## Part 4: Implementation Architecture

### Schema Design

The schema has seven tables organized in three groups:

**Containers and users:**

```typescript
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    name: v.string(),
  }),

  projects: defineTable({
    name: v.string(),
  }),
```

**Superclass table — the spine:**

```typescript
  recordings: defineTable({
    recordableType: recordableType,
    recordableId: recordableId,
    projectId: v.id("projects"),
    creatorId: v.id("users"),
  })
    .index("by_project", ["projectId"])
    .index("by_project_and_type", ["projectId", "recordableType"]),
```

**Recordable tables — append-only content:**

```typescript
  messages: defineTable({
    subject: v.string(),
    body: v.string(),
  }),

  comments: defineTable({
    content: v.string(),
  }),

  documents: defineTable({
    title: v.string(),
    body: v.string(),
  }),
```

**Event history — tracks every version:**

```typescript
  events: defineTable({
    recordingId: v.id("recordings"),
    recordableId: recordableId,
    creatorId: v.id("users"),
    action: v.union(v.literal("created"), v.literal("updated"), v.literal("copied")),
  }).index("by_recording", ["recordingId"]),
});
```

The shared validators are defined once and reused across the schema and all
function files:

```typescript
// The set of recordable type names — defined once, used everywhere.
const recordableType = v.union(
  v.literal("message"),
  v.literal("comment"),
  v.literal("document"),
);

// All possible recordable IDs — one per type table.
const recordableId = v.union(
  v.id("messages"),
  v.id("comments"),
  v.id("documents"),
);
```

### Design Principles

**1. The recordings table is the spine.**

Every query starts with recordings. Want all content in a project? Query
`recordings` by `projectId`. Want only messages? Filter on `recordableType`.
The recordable tables are never queried directly — they're reached by following
`recordableId` from a recording.

**2. Recordables are inert and immutable.**

A recordable has no foreign keys, no timestamps, no references to recordings
or projects or users. It is purely a bag of content-specific fields. As Jeff
Hardy puts it: "You got to think of the recordables as being pretty dumb.
They're just like, in the case of a message, it's literally just a title and
content. That's it. They have no connection to the outside world."

Recordables are never modified after creation. To edit content, you create a
new recordable and update the recording's pointer. The old recordable stays in
the database, untouched — available for version history, diffing, and rollback.

**3. Append-only is enforced by the write path.**

All writes to recordable tables go through a single `insertRecordable` helper
function. This helper only calls `ctx.db.insert()` — it has no capability to
patch or delete. There are no exported mutations for the `messages`, `comments`,
or `documents` tables. No function file exists for these tables at all. The
only way to get content into a recordable table is through `recordings.ts`,
and it can only append.

**4. Events are the change log.**

Every write that touches a recording's pointer — creating, editing, or
copying — also inserts an event. The events table records which recordable a
recording pointed to at each moment and who made the change. This is the same
pattern Basecamp uses for its document change history and "see what changed"
feature.

**5. Mutations are the consistency boundary.**

Creating a recording means inserting a recordable, a recording, and an event
in the same mutation. Because Convex mutations are transactional, there is no
window where a recording exists without its recordable, or an edit happens
without a corresponding event. This replaces Rails' reliance on callbacks and
`transaction` blocks.

**6. Type safety at the schema boundary.**

The `recordableType` field uses `v.union(v.literal(...))` — Convex validates
it at runtime and TypeScript checks it at compile time. The `recordableId`
field uses `v.union(v.id(...))` — each variant is typed to a specific table.
The pairing of type + ID is enforced by the `insertRecordable` helper, which
maps each type to its correct table and content shape.

### Function Organization

| File | Responsibility |
|---|---|
| `convex/recordings.ts` | All recording operations — record, edit, copy, list, get with history. The single entry point for all content. |
| `convex/projects.ts` | Project CRUD. Projects are the "buckets" — containers that hold recordings. |
| `convex/users.ts` | User lookup. Minimal — just enough to populate `creatorId`. |
| `convex/schema.ts` | Schema definition and shared validator constants (`recordableType`, `recordableId`). |

There are no `messages.ts`, `comments.ts`, or `documents.ts` function files.
This is intentional and structural — it means no code path exists that could
accidentally modify or delete a recordable. All content enters the system
through `recordings.ts`, which only appends to recordable tables.

### The Append-Only Write Path

The `insertRecordable` helper is the sole function that writes to recordable
tables. It maps each type to its table and content shape, and it only inserts:

```typescript
// Table name for each recordable type.
const RECORDABLE_TABLES = {
  message: "messages",
  comment: "comments",
  document: "documents",
} as const;

// The only write path to recordable tables. Insert-only — no patch, no delete.
async function insertRecordable(
  ctx: MutationCtx,
  recordableType: "message" | "comment" | "document",
  content: MessageContent | CommentContent | DocumentContent,
) {
  const table = RECORDABLE_TABLES[recordableType];
  return ctx.db.insert(table, content);
}
```

Every mutation in `recordings.ts` that needs to write content calls this
helper. The helper is not exported — it's a private function inside the module.
This is the Convex equivalent of Basecamp's discipline around recordable
immutability, but enforced structurally rather than by convention.

### Mutation Patterns

**Record: create a recording with its recordable.**

This is the Convex equivalent of Basecamp's `bucket.record(recordable, ...)`:

```typescript
export const record = mutation({
  args: {
    recordableType: recordableType,
    content: v.union(
      v.object({ subject: v.string(), body: v.string() }),  // message
      v.object({ content: v.string() }),                     // comment
      v.object({ title: v.string(), body: v.string() }),     // document
    ),
    projectId: v.id("projects"),
    creatorId: v.id("users"),
  },
  handler: async (ctx, { recordableType, content, projectId, creatorId }) => {
    // 1. Insert the recordable (append-only)
    const recordableId = await insertRecordable(ctx, recordableType, content);

    // 2. Insert the recording that references it
    const recordingId = await ctx.db.insert("recordings", {
      recordableType,
      recordableId,
      projectId,
      creatorId,
    });

    // 3. Log the creation event
    await ctx.db.insert("events", {
      recordingId,
      recordableId,
      creatorId,
      action: "created",
    });

    return recordingId;
  },
});
```

All three inserts happen in the same mutation — transactionally atomic. The
recordable is created first (to get its ID), then the recording, then the
event. If any step fails, all roll back.

**Edit: create a new version without touching the old one.**

This is the heart of the immutable recordables pattern. Editing never modifies
existing content — it creates a new recordable and swings the recording's
pointer:

```typescript
export const edit = mutation({
  args: {
    recordingId: v.id("recordings"),
    content: v.union(
      v.object({ subject: v.string(), body: v.string() }),
      v.object({ content: v.string() }),
      v.object({ title: v.string(), body: v.string() }),
    ),
    creatorId: v.id("users"),
  },
  handler: async (ctx, { recordingId, content, creatorId }) => {
    const recording = await ctx.db.get(recordingId);

    // 1. Insert a NEW recordable — the old one is untouched
    const newRecordableId = await insertRecordable(
      ctx, recording.recordableType, content
    );

    // 2. Swing the recording's pointer to the new version
    await ctx.db.patch(recordingId, { recordableId: newRecordableId });

    // 3. Log the edit event
    await ctx.db.insert("events", {
      recordingId,
      recordableId: newRecordableId,
      creatorId,
      action: "updated",
    });
  },
});
```

After this mutation, the recording points to the new content. The old
recordable still exists in the database — reachable through the events table.
This is exactly how Basecamp's document change log works: each event captures
a snapshot of the content at that moment in time.

**Copy: the cheap copy pattern.**

Copying creates a new recording that points to the same recordable. No content
is duplicated:

```typescript
export const copy = mutation({
  args: {
    recordingId: v.id("recordings"),
    destinationProjectId: v.id("projects"),
    creatorId: v.id("users"),
  },
  handler: async (ctx, { recordingId, destinationProjectId, creatorId }) => {
    const source = await ctx.db.get(recordingId);

    // New recording, SAME recordable — zero content duplication
    const newRecordingId = await ctx.db.insert("recordings", {
      recordableType: source.recordableType,
      recordableId: source.recordableId,
      projectId: destinationProjectId,
      creatorId,
    });

    await ctx.db.insert("events", {
      recordingId: newRecordingId,
      recordableId: source.recordableId,
      creatorId,
      action: "copied",
    });

    return newRecordingId;
  },
});
```

This is the efficiency Jeff Hardy highlights: "instead of actually copying the
content of a message, we just create a new recording row and point to the same
message recordable that already exists. We don't need to copy it at all, and
it's super fast and storage efficient." If a message gets copied 100 times,
there is still only one message recordable in the database.

### Query Patterns

**List recordings in a project, optionally filtered by type:**

```typescript
export const list = query({
  args: {
    projectId: v.id("projects"),
    recordableType: v.optional(recordableType),
  },
  handler: async (ctx, { projectId, recordableType }) => {
    const recordings = await ctx.db
      .query("recordings")
      .withIndex("by_project_and_type", (q) =>
        recordableType
          ? q.eq("projectId", projectId).eq("recordableType", recordableType)
          : q.eq("projectId", projectId)
      )
      .collect();

    return Promise.all(
      recordings.map(async (recording) => ({
        ...recording,
        recordable: await ctx.db.get(recording.recordableId),
      }))
    );
  },
});
```

This is the Convex equivalent of `bucket.recordings.messages` — a single
indexed query on the `recordings` table, then a fan-out to fetch each
recordable. The fan-out is explicit and deliberate.

**Get a recording with its full version history:**

```typescript
export const getWithHistory = query({
  args: { recordingId: v.id("recordings") },
  handler: async (ctx, { recordingId }) => {
    const recording = await ctx.db.get(recordingId);
    const recordable = await ctx.db.get(recording.recordableId);

    // Fetch all events for this recording — each one references
    // the recordable that was current at that moment
    const events = await ctx.db
      .query("events")
      .withIndex("by_recording", (q) => q.eq("recordingId", recordingId))
      .collect();

    const history = await Promise.all(
      events.map(async (event) => ({
        ...event,
        recordable: await ctx.db.get(event.recordableId),
      }))
    );

    return { recording, recordable, history };
  },
});
```

Each event in the history carries a reference to the recordable that was
current at that point. Because recordables are immutable, every version is
still exactly as it was when created — there is no risk of a historical
version being accidentally modified. This is how Basecamp implements "see what
changed" and "make this the current version."

### Rollback: Make a Previous Version Current

Because every old recordable still exists, rollback is a pointer swap:

```typescript
export const rollback = mutation({
  args: {
    recordingId: v.id("recordings"),
    eventId: v.id("events"),
    creatorId: v.id("users"),
  },
  handler: async (ctx, { recordingId, eventId, creatorId }) => {
    const event = await ctx.db.get(eventId);

    // Point the recording back to the old recordable
    await ctx.db.patch(recordingId, { recordableId: event.recordableId });

    // Log the rollback as a new event
    await ctx.db.insert("events", {
      recordingId,
      recordableId: event.recordableId,
      creatorId,
      action: "updated",
    });
  },
});
```

No content is recreated. The old recordable is already there. We just swing the
pointer back. As Jeff Hardy describes it: "When you click [make this the
current version], all we're going to do is update the recording record to point
to this version of the document instead of the current version."

---

## Why This Pattern Works in Convex

The delegated type pattern is a natural fit for Convex for a few reasons that
go beyond "you can express the same schema":

**Transactional mutations replace callbacks.** In Rails, creating a recording
with its recordable and event requires a `transaction` block, careful callback
ordering, and service objects like `Bucket::Recorder`. In Convex, a single
mutation is already atomic. The `record` mutation inserts three documents and
they either all succeed or none do. No ceremony required.

**Reactive queries replace cache invalidation.** In Basecamp, the Russian doll
caching pattern (`cache(recording) do...end`) is essential for performance.
In Convex, queries are reactive subscriptions. When a recording's pointer
swings to a new recordable, every component subscribed to that recording
re-renders with the new content. There is no cache key to compute, no cache to
expire, no stale data.

**Append-only is enforceable, not just conventional.** In Rails, the
immutability of recordables is a team discipline. Nothing in Active Record
prevents `message.update!(body: "oops")`. In Convex, the `insertRecordable`
helper is the only write path to content tables, and it only inserts. There
are no function files for recordable tables and no exported mutations that
could patch or delete content. The constraint is structural.

**Typed IDs make `ctx.db.get()` table-aware.** In Rails, `recordable_id` is an
integer and `recordable_type` is a string — the ORM uses both to locate the
record. In Convex, the table is encoded in the ID itself.
`ctx.db.get(recording.recordableId)` returns a document from the correct table
without needing a separate type column lookup. The `recordableType` field
exists for filtering and display, not for resolution.
