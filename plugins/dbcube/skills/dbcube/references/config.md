# Dbcube configuration (`dbcube.config.js`)

> Source of truth for the installed version: `node_modules/@dbcube/core/dist/index.d.ts`
> (config/engine). Verify there if an option isn't listed below.

A CommonJS file at the project root exporting a function that receives `config`
and calls **`config.set({ databases: { ... } })`**. There is **no**
`config.addDatabase()`.

```js
// dbcube.config.js
require("dotenv").config({ quiet: true });

module.exports = function (config) {
  config.set({
    databases: {
      // key = connection name → used by dbcube.database('<key>') and @database("<key>")
      myapp: {
        type: "mysql", // "mysql" | "postgres" | "sqlite" | "mongodb"
        config: {
          HOST: process.env.DBCUBE_MYAPP_HOST,
          USER: process.env.DBCUBE_MYAPP_USER,
          PASSWORD: process.env.DBCUBE_MYAPP_PASSWORD,
          DATABASE: process.env.DBCUBE_MYAPP_DATABASE,
          PORT: parseInt(process.env.DBCUBE_MYAPP_PORT, 10),
        },
        // optional:
        pool: { maxConnections: 5, minConnections: 2, acquireTimeoutMs: 3000, idleTimeoutMs: 3600000, sessionIdleTimeoutMs: 300000 },
        daemon: { requestTimeoutMs: 30000 },
      },
    },
  });
};
```

## Connection parameters (inside `config`)
| Key | Type | For | Notes |
|---|---|---|---|
| `DATABASE` | string | all | DB name; for SQLite the file name (stored as `.dbcube/<name>.db`) |
| `HOST` / `PORT` | string / number | mysql, postgres, mongodb | server host/port |
| `USER` / `PASSWORD` | string | mysql, postgres, mongodb | credentials |
| `URL` | string | cloud / Turso | full connection string (host, creds, TLS) |
| `AUTH_TOKEN` | string | Turso/libSQL | edge auth token |

## Connection pool
`pool: { maxConnections, minConnections, acquireTimeoutMs, idleTimeoutMs, sessionIdleTimeoutMs }`.
Defaults ≈ 5/2/3000/3600000 (SQLite 10/1), `sessionIdleTimeoutMs` 300000.
**Sizing:** serverless/poolers → small `maxConnections` (even 1); long-running
servers → higher; benchmarks → the same budget on both sides
(`{ maxConnections: 1, minConnections: 1 }`).
Rule for replicas: `replicas × maxConnections` must stay under the DB's limit.

**`sessionIdleTimeoutMs`** (default `300000` = 5 min) — server-side idle *session*
timeout: the database itself terminates a session left idle longer than this. It
bounds how long **orphaned connections** survive after an ungraceful client death
(e.g. a container that keeps crashing and restarting), which otherwise pile up
until the server runs out of connections. Applies to MySQL (`wait_timeout`),
PostgreSQL (`idle_session_timeout`, PG 14+) and MongoDB (`maxIdleTimeMS`); SQLite
is local and unaffected. Live pooled connections are unaffected — Postgres keeps
its shared connections alive with a keep-alive ping, and MySQL revalidates on
acquire. Set to `0` to disable and keep the server defaults. Requires
query-engine **v1.1.3+**.

## Cloud databases (URL + TLS) — same code, just config
TLS is on by default for managed hosts.

```js
// Supabase / Neon / RDS (PostgreSQL) — Supabase needs the connection pooler host
app: { type: "postgres", config: { URL: process.env.DATABASE_URL } }

// PlanetScale / Aiven (MySQL)
app: { type: "mysql", config: { URL: process.env.DATABASE_URL } }

// MongoDB Atlas
app: { type: "mongodb", config: { URL: process.env.MONGODB_URI } }

// Turso (libSQL / SQLite at the edge)
edge: { type: "sqlite", config: { URL: process.env.TURSO_URL, AUTH_TOKEN: process.env.TURSO_TOKEN } }
```
`TURSO_URL` looks like `libsql://<db>-<org>.turso.io`. For Postgres on Supabase
use the pooler host (the direct host is often IPv6-only). MongoDB transactions
require a replica set (Atlas has one; locally use `--replSet`).

## Multiple databases
Add more keys under `databases` and address each with `dbcube.database('<key>')`.
A transaction is scoped to ONE database/engine — there is no single transaction
across two different databases (use an outbox/saga pattern for that).
