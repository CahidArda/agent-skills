---
name: neon-realtime
description: >-
  Live SQL queries on Lakebase Postgres: subscribe to any read-only SELECT and
  receive its rows, then every committed change, in the browser, an agent, or
  a backend. Use when a user wants live or reactive data from their database —
  a UI that updates when rows change, collaborative or multi-user sync,
  live dashboards, feeds, chat, or instant optimistic writes — instead of
  polling or hand-rolling WebSocket fan-out. Covers enabling Realtime in
  neon.ts, sealing queries on the server, the React bindings
  (useLiveQuery), TanStack DB collections, Drizzle and Kysely adapters, SSR,
  optimistic mutations with awaitTxId, and debugging subscriptions. Triggers
  include "realtime", "live query", "live updates", "reactive app", "sync
  data to the client", "subscribe to changes", "push database changes to the
  browser", "useLiveQuery", "TanStack DB", "optimistic updates", "multiplayer",
  "port Supabase Realtime", "Neon Realtime", and "@neon/realtime".
metadata:
  parent: neon
  source: https://github.com/neondatabase/agent-skills/tree/main/skills/neon-realtime
---

**FIRST**: Use the parent `neon` skill for a Neon overview, getting started with Neon, Neon development best practices, and more.

If the `neon` skill is not installed, fetch it from https://neon.com/docs/ai/skills/neon/SKILL.md or install it with:

```bash
neon skills -s neon -y
```

# Neon Realtime

Currently in beta and available in `aws-us-east-2`, `aws-us-east-1`, `aws-eu-central-1`, and `aws-ap-southeast-1`.

Neon Realtime turns any read-only, deterministic SQL query into a live query. A subscription starts with the query's full result, then receives each committed transaction as an atomic batch. The live query engine runs inside Lakebase Postgres off logical replication, and the Live SQL API fans changes out to clients over one multiplexed WebSocket.

Use this skill to help the user enable Realtime, define and seal live queries in their backend, and bind them to a React or TanStack DB frontend. Deliver a working live query, a reactive UI wired to it, or a precise answer from the official Neon docs.

## When to Use

- **The UI should reflect database changes as they commit.** Feeds, chat, dashboards, inboxes, collaborative and multi-device views. Realtime replaces polling and replaces writing your own change fan-out.
- **They want instant local writes.** Apply an optimistic change, write through the backend, and drop the optimistic state when the transaction arrives on the subscription.
- **They are porting Supabase Realtime or a Data API app.** Replace channel or `postgres_changes` subscriptions with live queries sealed by the backend.
- **An agent needs to watch data.** A backend or agent runtime can subscribe directly with the secret, or receive a sealed query like a browser does.

Use the `neon-functions` skill instead when the app pushes its own messages rather than database state: presence, typing indicators, agent token streams, or a custom WebSocket/SSE protocol.

## What It Does

- **Live SQL** — Joins, aggregates, window functions, sub-queries, `ORDER BY` and `LIMIT`. Queries must be read-only and deterministic: no `now()`, no `gen_random_uuid()`.
- **Sealed queries** — The backend signs and encrypts the exact query, its parameters, and the database into a short-lived capability. Clients subscribe without seeing the SQL or holding the secret.
- **Consistent batches** — Each batch carries the transaction IDs (txids) it applied, so `awaitTxId()` confirms a write has synced.
- **Basic client bindings** — React (`RealtimeProvider`, `useLiveQuery`, `useRealtimeClient`; React 18 and 19) and a TanStack DB collection. Other frameworks build on the client's `subscribe()`.
- **Adapters** — Drizzle and Kysely builders become live queries with their row types intact.
- **Branches and scale-to-zero** — Realtime is enabled per branch, and child branches inherit it. Open subscriptions don't keep the compute awake.

## Availability

Check these before setting anything up:

- The project is in one of the regions above. Realtime is Neon only; it is not available on enterprise Lakebase.
- The beta is free for the Realtime service. Database compute woken by a new live query and egress are billed as usual.
- Row-level security is not supported. Authorize in the backend that seals queries (see [Rules](#rules)).
- The wire protocol is not stable. Use the SDK packages and never speak the WebSocket protocol directly.

## Setup

Realtime is part of `neon.ts` (see the `neon` skill for the branch-first workflow, `link`/`checkout`, and `neon.ts` basics):

```typescript
// neon.ts
import { defineConfig } from "@neon/config/v1";

export default defineConfig({
  realtime: true,
  // Or restrict browser origins:
  // realtime: { allowedOrigins: ["https://app.example.com"] },
});
```

```bash
neon config add realtime   # writes `realtime: true` (repeat --allowed-origin <origin> for the object form)
neon deploy                # alias for `neon config apply`; enables Realtime on the linked branch
```

`neon config status` and `neon config plan` show and diff the branch's Realtime state like any other `neon.ts` service. A new project can start with `neon init --realtime`. Without a `neon.ts`, enable a branch directly with `neon realtime enable` (`--project-id` and `--branch` select the target; `neon realtime disable` turns it off and discards the secret).

Tables need a stable replica identity. A primary key is enough; `REPLICA IDENTITY FULL` makes updates and deletes cheaper for the engine:

```sql
ALTER TABLE messages REPLICA IDENTITY FULL;
```

Install the SDK, plus the binding and adapter the app uses:

```bash
npm install @neon/realtime
npm install @neon/realtime-react        # React
npm install @neon/realtime-tanstack @tanstack/react-db   # TanStack DB
npm install @neon/realtime-drizzle      # or @neon/realtime-kysely
```

## Environment Variables

`neon deploy`, `link`, `checkout`, and `neon env pull` write these into `.env`/`.env.local` once Realtime is enabled (`neon env pull --services realtime` pulls only these):

| Variable                      | Where       | Meaning                                                |
| ----------------------------- | ----------- | ------------------------------------------------------ |
| `NEON_REALTIME_URL`           | Server, browser | Live SQL API WebSocket URL for `createRealtimeClient()` |
| `NEON_REALTIME_SECRET`        | Server only | Seals queries. Anyone holding it can seal any query     |
| `NEON_REALTIME_DATABASE_NAME` | Server      | The `db` passed to `createRealtime()`                   |

The browser needs only the URL. Expose it under the framework's public prefix (`VITE_NEON_REALTIME_URL`, `NEXT_PUBLIC_NEON_REALTIME_URL`). Never give the secret a public prefix. With `@neon/env`, `parseEnv(config)` returns a typed `env.realtime` (`url`, `secret`, `databaseName`).

## Prove It Works from the Terminal

Before writing app code, subscribe from the CLI. It resolves the branch's Realtime env the same way `neon env pull` does:

```bash
neon realtime subscribe "SELECT id, value FROM items"            # prints the full result on every commit
neon realtime subscribe --changes "SELECT id, value FROM items"  # prints reset and batch events instead
```

Insert a row from another terminal (`neon psql -- -c "INSERT INTO items (value) VALUES ('bar')"`) and the subscription updates. Add `--output json` for one NDJSON event per line. With `--changes`, `ORDER BY` and `LIMIT` queries can emit resets instead of batches.

## Build a Reactive App

### 1. Define and seal queries on the server

```typescript
// backend/realtime.ts — import from server code only
import { createRealtime } from "@neon/realtime/server";
import { drizzleAdapter } from "@neon/realtime-drizzle";

export const realtime = createRealtime({
  secret: process.env.NEON_REALTIME_SECRET!,
  db: process.env.NEON_REALTIME_DATABASE_NAME!,
  adapter: drizzleAdapter(), // or kyselyAdapter() from @neon/realtime-kysely; omit for rawSql()
});
```

Without an ORM, build queries with `rawSql<Row>("select id, body from messages where channel_id = $1", [channelId])` from `@neon/realtime/server`.

### 2. Expose the sealed query from an authorized handler

Seal per request, inside a route handler or server function that has verified the user:

```typescript
app.post("/api/messages/live", async (c) => {
  const user = await requireUser(c); // verified session, e.g. Neon Auth
  const { channelId } = await c.req.json();
  if (!Number.isInteger(channelId)) return c.json({ error: "invalid channelId" }, 400);
  const query = messagesQuery(user.id, channelId); // joins channel_members on user.id
  return c.json(await realtime.seal({ query }));
});
```

`seal()` is local crypto with no network call. The result is a bearer token valid for about 60 seconds.

### 3. Subscribe in the client

**React.** Create one client per app and wrap the tree in `RealtimeProvider`. `useLiveQuery` subscribes on mount, unsubscribes on unmount, and calls `refreshQuery` before the sealed query expires:

```tsx
import { useCallback } from "react";
import { createRealtimeClient, type SealedLiveQuery } from "@neon/realtime/client";
import { RealtimeProvider, useLiveQuery } from "@neon/realtime-react";

const client = createRealtimeClient({ url: import.meta.env.VITE_NEON_REALTIME_URL });
// <RealtimeProvider client={client}><App /></RealtimeProvider>

function Messages({ channelId, sealedQuery }: { channelId: number; sealedQuery: SealedLiveQuery<Message> }) {
  const refreshQuery = useCallback(() => fetchSealedMessages(channelId), [channelId]);
  const { data, status, error, utils } = useLiveQuery(sealedQuery, { refreshQuery });
  // data: readonly Message[] | undefined; status: "connecting" | "live" | "stale" | "error" | "closed"
}
```

**TanStack DB.** Sync one broader sealed query into a collection and derive local live queries (filters, sorts, joins) from it. The collection takes the client directly, so no provider is needed:

```typescript
import { createCollection } from "@tanstack/react-db";
import { realtimeCollectionOptions } from "@neon/realtime-tanstack";

const messages = createCollection(
  realtimeCollectionOptions({
    id: "messages",
    client,
    query: sealedInbox,
    refreshQuery: fetchSealedInbox,
    getKey: (m) => m.id, // a stable primary key, never the subscription rowId
  }),
);
```

**Other frameworks or plain TypeScript.** `client.subscribe(sealedQuery)` returns a subscription with `onChange(snapshot)`, `getState()`, `renew()`, `awaitTxId()`, and `unsubscribe()`. The React binding is a thin `useSyncExternalStore` wrapper over it, so port it for Vue, Svelte, or Solid rather than inventing a new pattern.

## Optimistic Writes

1. Apply the change to local optimistic state (React `useOptimistic`, or a TanStack DB collection mutation).
2. Write through the backend. Inside the write transaction, read `select pg_current_xact_id()::text as txid` and return the txid.
3. `await utils.awaitTxId(txid)` (React) or `await collection.utils.awaitTxId(txid)` (TanStack DB `onInsert`/`onUpdate`/`onDelete`).
4. The optimistic state drops after the synced rows already contain the change, so nothing flickers. On error or timeout, drop it and show the error.

The write needs an interactive transaction, so use a driver that supports one (for example `drizzle-orm/neon-serverless` with `Pool`, not `neon-http`).

## Server-Side Rendering

Per request, run the query and seal it in parallel (`Promise.all([query, realtime.seal({ query })])`), and pass both to the page. Give the rows to `useLiveQuery` as `initialData` (or seed and hydrate the TanStack DB collection). The rows render immediately as `stale`, and the first reset swaps in live rows. Create the client inside the React tree with `useState(() => createRealtimeClient(...))`. The server render never opens a WebSocket. Keep row shapes identical on both sides: `Date` survives React Server Component props and TanStack Start, but not plain `JSON.stringify`. Mark the module that holds the secret `server-only` (Next.js) or `*.server.ts` (TanStack Start).

## Rules

- **The secret stays on the server.** It belongs in the same trust tier as `DATABASE_URL`. Only `NEON_REALTIME_URL` and sealed queries reach the browser.
- **Authorize the query, not the connection.** Take the user ID from a verified session, filter or join on it inside the query, and validate every client-supplied parameter. Because the join is part of the live query, removing a membership revokes access immediately.
- **Keep sealed queries out of URLs, logs, and persistent storage.** They are bearer tokens until they expire.
- **`refreshQuery` must re-seal the same query with the same parameters.** A different parameter is a different query: seal it and subscribe again.
- **One client per app, one database per client.** The first sealed query binds the connection to its database.
- **Match result names to column names.** Use snake_case Drizzle keys or `getTableColumns()`, and alias computed columns. Drizzle and Kysely result transformers (such as Kysely's `CamelCasePlugin`) don't apply to live rows.
- **Know the decoded types.** `timestamptz` arrives as `Date`, `json`/`jsonb` parsed, `int8`/`numeric` as `string`. With Drizzle, pass `parsers: drizzleParsers` from `@neon/realtime-drizzle/client` to match Drizzle's date modes.

## Debugging

- **Isolate the query** with `neon realtime subscribe "<sql>"`. If it fails there, the problem is the query or the branch, not the app.
- **Client diagnostics**: `createRealtimeClient({ url, logLevel: "debug" })`. Levels are `silent` (default), `error`, `warn`, `info`, and `debug`. Pass `logger: (entry) => …` and match on `entry.event` (for example `connection_lost`, `subscription_failed`), not on `message`.
- **Full database errors**: `createRealtime({ ..., debugMode: process.env.NODE_ENV === "development" })`. Errors then carry Postgres's message and `sqlState`. Never enable it in production; it exposes schema, table, and column names to subscribers.
- **Error states**: a subscription in `error` never recovers on its own. Unsubscribe and subscribe again with a fresh sealed query. A connection-level failure (fatal protocol error, a transaction over the 16 MiB client limit) fails every subscription; create a new client.
- **Stuck on `stale`**: the sealed query expired and renewal is failing. Check that `refreshQuery` is passed, authenticates, and seals the identical query.

## Neon Documentation

Realtime is in beta and evolving quickly, so verify against the official docs. Append `.md` to any docs URL to fetch it as markdown, and look pages up in https://neon.com/docs/llms.txt rather than guessing URLs.

## Further Reading

- https://neon.com/docs/realtime/overview.md
- https://neon.com/docs/realtime/quickstart.md
- https://neon.com/docs/realtime/how-it-works.md
- https://neon.com/docs/realtime/guides/queries.md
- https://neon.com/docs/realtime/guides/subscribing.md
- https://neon.com/docs/realtime/guides/reactivity.md
- https://neon.com/docs/realtime/guides/mutations.md
- https://neon.com/docs/realtime/guides/ssr.md
- https://neon.com/docs/realtime/sdks/typescript.md
- https://neon.com/docs/realtime/examples/nextjs.md
- https://neon.com/docs/realtime/examples/tanstack.md
