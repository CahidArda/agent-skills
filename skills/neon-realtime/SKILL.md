---
name: neon-realtime
description: >-
  Neon Realtime makes any SQL query live. Build realtime, reactive apps and agents on Lakebase Postgres with live SQL queries and reactivity primitives. Use when app UI or agent state must update live from database rows (dashboards, feeds, collaboration, multi-user or multi-device sync), for optimistic mutations, or to replace polling or other realtime sync systems. Triggers include "live query", "subscribe", "sync engine", "useLiveQuery", "TanStack DB", "optimistic updates", "@neon/realtime". For push that doesn't come from database rows (presence, typing, LLM streams), use neon-functions.
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

Neon's Realtime makes any SQL query live. Subscribe in realtime to your Lakebase Postgres. Develop apps and agents with end-to-end reactivity using type-safe live queries and optimistic mutations.

Realtime is new and in beta, so its APIs aren't in your training data. For working code, fetch the guide linked in each section rather than guessing.

## How It Works

1. In trusted backend code, **define** a live query (raw SQL, Drizzle or Kysely), **seal** it with `NEON_REALTIME_SECRET` and **expose** the sealed query from an authenticated endpoint or server function. The client can't read or change a sealed query's SQL or parameters.
2. The client **subscribes** with the sealed query. It receives the query's full initial result, then batches of changes as transactions commit, each applied all at once. Agents and workers can also subscribe directly from trusted code.
3. **Writes** go through your own API. Return the write's `txid`, and `awaitTxId()` clears the optimistic state once the write has synced.

A live query can be any read-only SQL over your existing schema, including joins, aggregates, ordering and limits. Realtime keeps results consistent but doesn't guarantee delivery of every change event.

Coming from Electric, live SQL queries replace shapes and `realtimeCollectionOptions()` replaces `electricCollectionOptions()`.

## Setup

- **New app:** `neon bootstrap my-app --template realtime-react`, then `neon deploy` in `my-app`.
- **Enable** it on the linked branch with [`neon realtime enable --allowed-origin <app origin>`](https://neon.com/docs/cli/realtime.md), or declare [`preview: { realtime: true }`](https://neon.com/docs/reference/neon-ts.md#preview-realtime) in `neon.ts` (`neon config add realtime`) and run `neon deploy`. Child branches inherit it. It isn't available on Claimable projects until they're claimed, or outside the [supported regions](https://neon.com/docs/realtime/overview.md#which-regions-is-it-enabled-in). `REPLICA IDENTITY FULL` on the tables you query lowers sync latency.
- **Environment:** `neon env pull --service postgres --service realtime` writes `DATABASE_URL`, `NEON_REALTIME_URL` and `NEON_REALTIME_SECRET`. Expose only the URL to the browser, under your framework's public prefix (for example `VITE_NEON_REALTIME_URL`). `url` options take `NEON_REALTIME_URL`, and `db` takes the database name from `DATABASE_URL`.
- **Install** `@neon/realtime` and the bindings you use: `@neon/realtime-react`, `@neon/realtime-tanstack`, `@neon/realtime-drizzle` or `@neon/realtime-kysely`. Use these SDKs rather than the WebSocket protocol, which is still evolving.

## Define and Seal Queries

`createRealtime({ secret, db, adapter })` from `@neon/realtime/server` creates the server, and `realtime.seal({ query })` seals a query. Write raw SQL with `rawSql` and `pgParam` (no adapter needed), or pass `drizzleAdapter()` or `kyselyAdapter()` as `adapter`. With any other ORM, use raw SQL. Code: [Defining queries](https://neon.com/docs/realtime/guides/queries.md).

- **Read-only and deterministic.** A raw query is a single `SELECT` with no `;`, and can't call `now()` or `gen_random_uuid()`, so pass such values in as parameters. Use `$1` placeholders, and wrap every value except strings, numbers, booleans, bigints and null in a `pgParam` helper, such as `pgParam.timestamptz(date)`. With Drizzle, each selected key must match its column name.
- **Authorize in the query.** Check the session before sealing, and join through your membership tables so revoked access drops out of the results. Realtime works with any web framework auth, including Neon Auth. It doesn't support RLS yet (but may in future).
- **A sealed query is a bearer token.** Keep it out of URLs, logs and persistent storage, and never fill its parameters from unchecked input.
- **Sealed queries expire**. Pass `refreshQuery` to the bindings or call `renew()`, sealing the same query and parameters, or the subscription errors. New parameter values need a new sealed query and subscription. In Next.js, renew from a Route Handler, because server actions run one at a time.

## Subscribe

Client code creates one client with `createRealtimeClient({ url, parsers })` from `@neon/realtime/client` (with Drizzle, spread `drizzleParsers` from `@neon/realtime-drizzle/client` into `parsers`), then uses the highest layer that fits. Code: [Client-side reactivity](https://neon.com/docs/realtime/guides/reactivity.md) and [Subscribing to queries](https://neon.com/docs/realtime/guides/subscribing.md).

| Where | API |
| --- | --- |
| React | `<RealtimeProvider client={client}>` and `useLiveQuery(sealed, { refreshQuery, initialData })` from `@neon/realtime-react` |
| TanStack DB, to sync one broad query into many local views | `realtimeCollectionOptions` from `@neon/realtime-tanstack` |
| Other frameworks | `client.subscribe(sealed)`, with `getSnapshot`, `onChange` and `onStateChange` |
| Agents and workers | Add `url` to `createRealtime`, then `await realtime.subscribe(query)`, which seals in-process |

- **One database per client.** The first sealed query binds the connection to its `db`.
- **Key rows by primary key**, including TanStack DB's `getKey`. Never use `rowId`, which every reset invalidates.
- **`error` is final**, even after `renew()`. Fetch a fresh sealed query and subscribe again, after creating a new client if the connection failed. For direct subscriptions, watch `onStateChange` and resubscribe. `useLiveQuery()` reports them through `status` rather than throwing or suspending, so check it.
- **On sign-out, unmount subscribers**, because failed renewals leave them `stale` and retrying. With a TanStack `DbClient`, do a full page load so the previous user's rows go.

## Optimistic Writes

Code: [Optimistic mutations](https://neon.com/docs/realtime/guides/mutations.md), for each layer.

- Select `pg_current_xact_id()::text` inside the write transaction and return it as the `txid`. That needs a driver with interactive transactions, such as `neon-serverless`.
- Always pass `awaitTxId()` a timeout, because it doesn't reject when the subscription errors.
- A write that changes no rows has no txid, so skip `awaitTxId()` and let the optimistic state drop.

## Server-Side Rendering

Code: [Server-side rendering](https://neon.com/docs/realtime/guides/ssr.md) for Next.js and TanStack Start.

- Create the client inside the React tree (for example in `useState`), and render `initialData` as live rows without a loading state.
- Keep row shapes identical on both sides. `JSON.stringify` turns dates into strings, and `int8` and `numeric` arrive as strings.
- Never share a TanStack `DbClient` between requests.

## Verify and Debug

[`neon subscribe "SELECT ..."`](https://neon.com/docs/cli/subscribe.md) prints a query's full result when it starts and again after every change, so you can check a query before wiring it into an app. For full database errors in development only, set `debugMode` on `createRealtime`.

## More Docs

| Task | Page |
| --- | --- |
| Beta scope, regions, pricing, FAQ, other systems | [Overview](https://neon.com/docs/realtime/overview.md) |
| Architecture and consistency | [How it works](https://neon.com/docs/realtime/how-it-works.md) |
| A first live query | [Quickstart](https://neon.com/docs/realtime/quickstart.md) |
| Every SDK export | [TypeScript SDK reference](https://neon.com/docs/realtime/sdks/typescript.md) |
| Full-stack apps with auth and SSR | [Next.js](https://neon.com/docs/realtime/examples/nextjs.md), [TanStack](https://neon.com/docs/realtime/examples/tanstack.md) |
| Live agent workflows | [Mastra](https://neon.com/docs/realtime/examples/mastra.md) |
| ORM adapters | [Drizzle](https://neon.com/docs/realtime/examples/drizzle.md), [Kysely](https://neon.com/docs/realtime/examples/kysely.md) |
