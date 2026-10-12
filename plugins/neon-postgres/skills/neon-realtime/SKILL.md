---
name: neon-realtime
description: >-
  Neon Realtime makes any SQL query live. Build realtime, reactive apps and agents. Subscribe in realtime to Lakebase Postgres with type-safe live queries and optimistic mutations. Use when an app or agent needs data that updates live (realtime reactivity, reactive UIs, live dashboards, feeds, chat, collaboration, multi-user sync, multi-device, optimistic mutations) and for apps and agents that need instant reads and writes for fast, modern AX and DX. Neon Realtime replaces the need for polling or a separate realtime system. Triggers include "sync", "realtime", "reactivity", "live query", "subscribe", "subscriptions", "sync engine", "useLiveQuery", "TanStack DB", "optimistic updates", "@neon/realtime". For non-database push (presence, typing, LLM streams) use neon-functions.
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

Realtime is new and in beta, so its APIs aren't in your training data. Work from this skill and the docs it links rather than guessing.

## What it is

Realtime is a live SQL based sync engine, with (optional) higher-level reactivity primitives for client-side development. Realtime offers arbitrary live SQL with full reactivity on Lakebase Postgres. It allows you to build realtime, reactive apps and agents:

- without using a second or bespoke realtime system
- without changing your schema, queries, ORM or reactivity system
- without contorting your application around the limitations of your sync engine

Realtime has zero impact on the write-latency, write-throughput or write-compute of your primary database, and live queries don't use up database connections.

## Compared with alternatives

The [overview FAQ](https://neon.com/docs/realtime/overview.md) compares Realtime with other systems, noting "We don't speak for other systems":

- **Supabase Realtime** is a pubsub system with replay. Its subscriptions "can filter on column values but can't join, aggregate, order, limit or run arbitrary queries." Neon Realtime is a sync engine that supports arbitrary live SQL queries, including joins, aggregates, ordering and limits.
- **Convex**: similar end-to-end reactivity, but Realtime runs on standard Postgres with standard web frameworks and build tooling, and "uses delta-sync rather than query-invalidation, which is more efficient over the wire."
- **Zero**: Realtime "is built into Lakebase Postgres and the Neon platform and provides similar sync and reactivity on standard SQL."
- **Electric**: Realtime "is a drop-in replacement for Electric's role in your stack." Live SQL queries replace shapes, and `realtimeCollectionOptions()` replaces `electricCollectionOptions()`.

Realtime serves live query results to apps and agents. It ensures eventual consistency of live query results. It does not guarantee every-change-event delivery. Push that doesn't come from database rows (presence, typing, LLM streams) belongs on a Neon Function (`neon-functions` skill).

## How it works

1. Enable Realtime on the branch. Tables need a stable replica identity: a primary key is enough; setting `REPLICA IDENTITY FULL` on a table makes live queries faster (lower sync latency).
2. In trusted backend code, **define** a live query (raw SQL, Drizzle or Kysely), **seal** it with `NEON_REALTIME_SECRET` and **expose** the sealed query from an authenticated endpoint or server function.
3. The client subscribes through the Live SQL API. It receives the query's full initial result, then batches of changes as transactions commit, each applied all at once. The client can't read or change a sealed query's SQL or parameters.
4. Writes go through your own API. Return the write's `txid`, and `awaitTxId()` clears the optimistic state once the write has synced.

## Packages

| Package | Key APIs |
| --- | --- |
| `@neon/realtime/server` | `createRealtime({ secret, db, adapter })`, `rawSql`, `pgParam`, `realtime.seal({ query })`. Adding `url` gives a direct server, whose async `realtime.subscribe(query)` runs in trusted code such as agents and workers. |
| `@neon/realtime/client` | `createRealtimeClient({ url, parsers })` and the synchronous `client.subscribe(sealed)`, whose subscription has `onChange`, `onStateChange`, `getSnapshot`, `renew` and `awaitTxId` |
| `@neon/realtime-react` | `RealtimeProvider`, `useLiveQuery(sealed, { refreshQuery, initialData })`, `useRealtimeClient` |
| `@neon/realtime-tanstack` | `realtimeCollectionOptions` for TanStack DB collections |
| `@neon/realtime-drizzle` | `drizzleAdapter()` for the `adapter` option. Spread `drizzleParsers` from `@neon/realtime-drizzle/client` into the client's `parsers`. |
| `@neon/realtime-kysely` | `kyselyAdapter()` for the `adapter` option |

Raw SQL needs no adapter. Use these SDKs rather than the WebSocket protocol, which is still evolving. The [TypeScript SDK reference](https://neon.com/docs/realtime/sdks/typescript.md) covers them all.

## Setup

- Enable Realtime with `neon realtime enable --allowed-origin <app origin>` on the linked branch, or declare `preview: { realtime: true }` in `neon.ts` (`neon config add realtime`) and run `neon deploy`. Without an allowed origin, a first enable allows every origin. Child branches inherit Realtime. It isn't available on Claimable projects until they're claimed, or outside supported regions.
- `neon env pull --service postgres --service realtime` writes `DATABASE_URL`, `NEON_REALTIME_URL` and `NEON_REALTIME_SECRET`. Expose only the URL to the browser, under your framework's public prefix (for example `VITE_NEON_REALTIME_URL`). The `db` option is the database name from `DATABASE_URL`.
- To start from a working app, run `neon bootstrap my-app --template realtime-react`, then `neon deploy` in `my-app`.

See the [quickstart](https://neon.com/docs/realtime/quickstart.md), [`neon realtime`](https://neon.com/docs/cli/realtime.md) and [`preview.realtime`](https://neon.com/docs/reference/neon-ts.md#preview-realtime).

## Rules

These are the mistakes that break generated code. The linked guides explain each one.

- **Live queries are read-only and deterministic.** A raw query is a single `SELECT` with no `;`. It can't call `now()` or `gen_random_uuid()`, so pass such values in as parameters. Use `$1` placeholders and never interpolate user input. Every value except strings, numbers, booleans, bigints and null needs a `pgParam` helper, such as `pgParam.timestamptz(date)`. With Drizzle, each selected key must match its column name. ([queries](https://neon.com/docs/realtime/guides/queries.md#defining-live-queries))
- **Authorize in the query.** Check the session before sealing, and join through your membership tables so the join is part of the live query and revoked access drops out of the results. Realtime doesn't support RLS yet. ([queries](https://neon.com/docs/realtime/guides/queries.md#authorization))
- **A sealed query is a bearer token.** Anyone holding one can subscribe until it expires, so send it over HTTPS and keep it out of URLs, logs and persistent storage. Seal only in trusted backend code, and never fill parameters from untrusted input without checking them.
- **Sealed queries expire** (currently after 60 seconds). Pass `refreshQuery` to the bindings, or call `renew()` with the same query. Different parameter values make a different query, so fetch a new sealed query for them. ([queries](https://neon.com/docs/realtime/guides/queries.md#renewing-sealed-queries))
- **One database per client.** The first sealed query binds the connection, so every subscription on a client must target the same `db`.
- **`error` is final.** A subscription in `error` doesn't recover and `renew()` can't bring it back, so fetch a fresh sealed query and subscribe again. After a connection-level failure, create a new client first. `useLiveQuery()` reports subscription errors through `status` rather than throwing or suspending, so check it, and unmount subscribers on sign-out. ([subscribing](https://neon.com/docs/realtime/guides/subscribing.md#handle-errors))
- **Optimistic writes need the txid.** Select `pg_current_xact_id()::text` inside the write transaction, which needs a driver with interactive transactions such as `neon-serverless`. Always pass `awaitTxId()` a timeout, because it doesn't reject when the subscription errors. Handle the `unchanged` response, which has no txid. ([mutations](https://neon.com/docs/realtime/guides/mutations.md#returning-the-txid-from-your-backend))
- **Key rows by primary key**, never by `rowId`, including TanStack DB's `getKey`.
- **SSR:** create the client inside the React tree (for example in `useState`), render `initialData` as live rows without a loading state, and keep row shapes identical on both sides. `JSON.stringify` turns dates into strings, and `int8` and `numeric` arrive as strings. Never share a TanStack `DbClient` between requests. ([SSR](https://neon.com/docs/realtime/guides/ssr.md))
- **`debugMode` is a development-only `createRealtime` option**, because it exposes full database errors. ([subscribing](https://neon.com/docs/realtime/guides/subscribing.md#show-full-database-errors))

## Verify

`neon subscribe "SELECT ..."` prints a query's full result when it starts and again after every change. `--changes` prints the raw reset and batch events instead, and `-o json` prints newline-delimited JSON. Use it to check a query before wiring it into an app. See [`neon subscribe`](https://neon.com/docs/cli/subscribe.md).

## Docs

Fetch the page for the task rather than guessing at APIs. For working code, start from the guides and examples.

| Task | Page |
| --- | --- |
| Beta scope, regions, pricing, FAQ | [Overview](https://neon.com/docs/realtime/overview.md) |
| Architecture, consistency, packages | [How it works](https://neon.com/docs/realtime/how-it-works.md) |
| Enable Realtime and run a first live query | [Quickstart](https://neon.com/docs/realtime/quickstart.md) |
| Define, parameterize, authorize, seal and expose queries | [Defining queries](https://neon.com/docs/realtime/guides/queries.md) |
| Low-level client, direct subscriptions, renewal, errors, debugging | [Subscribing to queries](https://neon.com/docs/realtime/guides/subscribing.md) |
| React and TanStack DB bindings | [Client-side reactivity](https://neon.com/docs/realtime/guides/reactivity.md) |
| Instant local writes with `awaitTxId` | [Optimistic mutations](https://neon.com/docs/realtime/guides/mutations.md) |
| Next.js and TanStack Start SSR | [Server-side rendering](https://neon.com/docs/realtime/guides/ssr.md) |
| Full-stack apps with auth and SSR | [Next.js](https://neon.com/docs/realtime/examples/nextjs.md), [TanStack](https://neon.com/docs/realtime/examples/tanstack.md) |
| Live agent workflows | [Mastra](https://neon.com/docs/realtime/examples/mastra.md) |
| ORM adapters | [Drizzle](https://neon.com/docs/realtime/examples/drizzle.md), [Kysely](https://neon.com/docs/realtime/examples/kysely.md) |
