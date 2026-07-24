# para/aether_db

**DB-backed, revocable sessions for [para/aether](https://github.com/noeta-lang/para-aether).** The stored upgrade to `std.session`'s stateless `CookieSessions`.

`std.session` carries the whole session in a signed cookie — no server state, correct under `serve --parallel`, nothing to configure. That is the right default, and most apps never need more. Two things it cannot do, by construction:

- **Grow past ~4 KB.** A browser silently drops a cookie over about 4 KB, so a stateless session has a hard ceiling. `std.session` errors when you hit it rather than losing state quietly — and that error is the signal to move here.
- **Revoke.** A signed cookie is valid until it expires; there is nothing server-side to invalidate, so a stolen token is stolen until expiry and a "log out everywhere" button is a lie. A stored session's authoritative copy lives in a row, so deleting the row is a **real** logout: a copy of the id taken earlier no longer resolves to anything.

`para/aether_db` keeps only a small opaque id in the cookie and the data in a `sessions` row. It implements para/aether's `SessionStore` over the **same** `std.session` `Session`, so `DbSessions` and `CookieSessions` are interchangeable at a `dyn SessionStore` call site — a handler written against one moves to the other without a line changing.

## What it provides

One pure-Noeta module, `para.aether_db`:

- **`DbSessions`** — a `SessionStore` implementation over a [para/db](https://github.com/noeta-lang/para-db) `Connection`: `DbSessions.new(conn, cookie_name, max_age_seconds, secure)`.
- **`migrations/`** — the one SQL migration the store needs (`sessions`: `id TEXT PK, data TEXT, expires_at INTEGER`), plain SQL that runs verbatim on both drivers para/db ships.

## Installation

The store lives in its own companion package, not in para/aether and not in para/db, on purpose:

- **Not in para/db** — a database package must not depend on the web framework.
- **Not in aether-core** — that would make para/db's native SQLite/Postgres footprint mandatory for *every* aether app, including the ones happy with cookie sessions.

So an app reaches for stored sessions by adding `para/aether_db` to the `para` scope it already lists aether (and para/db) under:

```toml
[dependencies]
para = [
    { version = "^0.1", package = "para/aether" },
    { version = "^0.1", package = "para/db" },
    { version = "^0.1", package = "para/aether_db" },   # <- add this
]

[trust]
native = ["para/db"]   # para/db's native driver; para/aether_db itself is pure Noeta
```

Discoverability is that dependency plus this doc — an app that references `para.aether_db.DbSessions` without the package gets an ordinary unresolved-import error. There is deliberately no bespoke compiler diagnostic: there is no central hook for one, and the import error already names the missing symbol.

## Usage

**1. Migrate.** The store needs one table. Copy the migration (or point `conn.migrate` at this package's `migrations/`):

```
migrations/20260722120000_create_sessions.sql   # id TEXT PK, data TEXT, expires_at INTEGER
```

```noeta ignore
conn = db.connect("sqlite:app.db")
conn.migrate("migrations")
```

**2. Bind the store** under the `SessionStore` interface, and inject it into handlers:

```noeta ignore
use para.aether.{App, Get, Post, SessionStore}
use para.aether_db.DbSessions
use std.http.server
use std.http.{Request, Response}

app.bind("SessionStore", DbSessions.new(conn, "sid", 3600, true))
//                                    conn ─┘  │      │     └ Secure (true behind https)
//                             cookie name ────┘      └ max-age seconds

class Account {
    #[Post("/login")]
    fn login(req: Request, store: dyn SessionStore): Response {
        s = store.open(req).set("user", "42")
        return store.attach(server.response(200, "hi"), s)
    }

    #[Post("/logout")]
    fn logout(req: Request, store: dyn SessionStore): Response {
        s = store.open(req).clear()                       // empties the session
        return store.attach(server.response(200, "bye"), s)  // → DELETEs the row, expires the cookie
    }
}
```

## How it behaves

| Handler does | `attach` does |
|---|---|
| reads only (`open`, no `set`/`remove`/`clear`) | nothing — no row write, no `Set-Cookie` (a read is free) |
| writes (`set`/`remove`) | UPSERTs the row (data as JSON, fresh expiry) under its id — minted on the first write — and sets the id cookie |
| `clear()` | **DELETEs** the row and expires the cookie — a revocable logout |

The cookie is `HttpOnly`, `SameSite=Lax`, `Path=/`, and `Secure` per the `secure` argument (`true` in production; `false` only for a plain-http localhost — the same out-loud choice `CookieSessions` asks for, because both wrong answers fail silently). The id is 16 crypto-random bytes (128 bits), hex-encoded — an unguessable **key**, never a signed payload: the row is the source of truth, so the id carries no data and needs no verification.

Expiry is compared in-app against `datetime.now()` (a bound parameter), not in SQL, so the store's one statement runs unchanged on SQLite and PostgreSQL alike.

## Examples

- [`examples/aether-sessions-db/`](examples/aether-sessions-db) — the runnable end-to-end version: login, a stored round-trip, and a logout that deletes the row.
- [`examples/aether-db-demo/`](examples/aether-db-demo) — an aether app composing para/db storage.

## Requirements

This package is pure Noeta, but it depends on para/db, whose native driver is compiled locally by the consumer's toolchain — so `cargo` and a Rust toolchain (1.95+) must be on `PATH` to build an app that uses it.

## Development

Each directory under `examples/` is its own small package depending on this repo by path (plus its sibling repos by git); run `noeta check` / `noeta test` there. See [AGENTS.md](AGENTS.md) for the repo layout and the toolchain environment the examples need.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
