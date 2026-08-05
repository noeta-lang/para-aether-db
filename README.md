# para/aether_db

**DB-backed, revocable sessions for [para/aether](https://github.com/noeta-lang/para-aether).** The stored upgrade to `std.session`'s stateless `CookieSessions`.

`std.session` carries the whole session in a signed cookie — no server state, correct under `serve --parallel`, nothing to configure. That is the right default, and most apps never need more. Two things it cannot do, by construction:

- **Grow past ~4 KB.** A browser silently drops a cookie over about 4 KB, so a stateless session has a hard ceiling. `std.session` errors when you hit it rather than losing state quietly — and that error is the signal to move here.
- **Revoke.** A signed cookie is valid until it expires; there is nothing server-side to invalidate, so a stolen token is stolen until expiry and a "log out everywhere" button is a lie. A stored session's authoritative copy lives in a row, so deleting the row is a **real** logout: a copy of the id taken earlier no longer resolves to anything.

`para/aether_db` keeps only a small opaque id in the cookie and the data in a `sessions` row. It implements para/aether's `SessionStore` over the **same** `std.session` `Session`, so `DbSessions` and `CookieSessions` are interchangeable at a `dyn SessionStore` call site — a handler written against one moves to the other without a line changing.

## What it provides

One pure-Noeta module, `para.aether_db`:

| symbol | kind | purpose |
| --- | --- | --- |
| `DbSessions` | class, `impl SessionStore` | the stored session store over a [para/db](https://github.com/noeta-lang/para-db) `Connection` |
| `DbSessions.new(conn, cookie_name, max_age, secure)` | constructor | build a store — see [Configuration](#configuration--the-four-knobs-dbsessionsnew-takes) |
| `open(req: Request): Session` | `SessionStore` method | the session the request's id cookie names, or an empty one |
| `attach(resp: Response, session: Session): Response` | `SessionStore` method | persist the session onto the response — row write and/or `Set-Cookie` as needed |
| [`migrations/`](migrations) | SQL | the one `sessions`-table migration the store needs, portable across both para/db drivers |

## Installation

The store lives in its own companion package, not in para/aether and not in para/db, on purpose:

- **Not in para/db** — a database package must not depend on the web framework.
- **Not in aether-core** — that would make para/db's native SQLite/Postgres footprint mandatory for *every* aether app, including the ones happy with cookie sessions.

So an app reaches for stored sessions by adding `para/aether_db` to the `para` scope it already lists aether (and para/db) under:

```toml
[dependencies]
para = [
    { version = "^0.4", package = "para/aether" },
    { version = "^0.5", package = "para/db" },
    { version = "^0.3", package = "para/aether_db" },   # <- add this
]

[trust]
native = ["para/db"]   # para/db's native driver; para/aether_db itself is pure Noeta
```

Discoverability is that dependency plus this doc — an app that references `para.aether_db.DbSessions` without the package gets an ordinary unresolved-import error. There is deliberately no bespoke compiler diagnostic: there is no central hook for one, and the import error already names the missing symbol.

## Usage

**1. Migrate.** The store needs one table. Copy [`migrations/20260722120000_create_sessions.sql`](migrations/20260722120000_create_sessions.sql) into your app's `migrations/` directory (or point `conn.migrate` at a checkout of this package), then apply it — at boot is fine, `conn.migrate` is a no-op when up to date:

```noeta ignore
use para.db

conn = db.connect("sqlite:app.db")   // or postgres://… — any dsn db.connect accepts
conn.migrate("migrations")
```

**2. Bind the store** under the `SessionStore` interface, and let aether's DI inject it — every handler parameter typed `dyn SessionStore` receives it:

```noeta ignore
use para.aether.{App, Get, Post, SessionStore}
use para.aether_db.DbSessions
use std.http.server
use std.http.{Request, Response}

class Account {
    fn new(): Account {
        return Account {}
    }

    #[Get("/whoami")]
    fn whoami(req: Request, store: dyn SessionStore): Response {
        s = store.open(req)
        who = s.get("user") ?? "anonymous"
        return store.attach(server.response(200, "you are " ~ who), s)
    }

    #[Post("/login")]
    fn login(req: Request, store: dyn SessionStore): Response {
        s = store.open(req).set("user", "ada")
        return store.attach(server.response(200, "logged in"), s)
    }

    #[Post("/logout")]
    fn logout(req: Request, store: dyn SessionStore): Response {
        s = store.open(req).clear()                          // empties the session
        return store.attach(server.response(200, "bye"), s)  // → DELETEs the row, expires the cookie
    }
}

app = App.new()
app.register("Account", Account.new())
app.bind("SessionStore", DbSessions.new(conn, "sid", 3600, true))
//                                    conn ─┘  │      │     └ Secure (true behind https)
//                             cookie name ────┘      └ max-age seconds
app.discover()
```

**3. Serve.** A deployed app hands aether's dispatch core to the server: `server.serve(port, app.serve_request)`. The handlers are identical whether requests arrive from a live socket or from a test (see [Testing](#testing--session-round-trips-without-a-socket)).

The handler pattern is always the same round-trip: `open` the request's session, read or write it, `attach` it to the response you return. `attach` looks at what happened and does the minimum — nothing for a pure read, an UPSERT for a write, a DELETE for a `clear()`.

## Configuration — the four knobs `DbSessions.new` takes

| argument | type | meaning |
| --- | --- | --- |
| `conn` | `Connection` | a live para/db connection — SQLite and PostgreSQL both work; the store's SQL runs unchanged on either driver |
| `cookie_name` | `string` | the cookie the opaque id rides in (e.g. `"sid"`) |
| `max_age` | `int` | session lifetime in seconds — used for both the row's `expires_at` and the cookie's `Max-Age` |
| `secure` | `bool` | the cookie's `Secure` attribute |

The lifetime window slides on **writes**, not reads: every `set` that reaches `attach` UPSERTs the row with a fresh expiry (`now + max_age`) and re-sets the cookie's `Max-Age`, while a pure read leaves both untouched. A session only read for `max_age` seconds therefore expires.

> [!WARNING]
> `secure` is an out-loud choice, the same one `CookieSessions` asks for, because both wrong answers fail silently: `true` on a plain-http localhost and the browser never sends the cookie back; `false` in production and the id crosses the network unprotected. Pass `true` behind https; `false` only for plain-http local development.

## The `sessions` table — one portable migration

The whole schema is one table, shipped as [`migrations/20260722120000_create_sessions.sql`](migrations/20260722120000_create_sessions.sql):

| column | type | holds |
| --- | --- | --- |
| `id` | `TEXT PRIMARY KEY` | the opaque id — the cookie's entire payload, a random key, never a signed blob |
| `data` | `TEXT NOT NULL` | the session data as a small JSON object |
| `expires_at` | `INTEGER NOT NULL` | absolute expiry in unix seconds; indexed (`sessions_expires_at`), since every read filters on it |

The DDL is deliberately portable — `TEXT`, `INTEGER`, and `PRIMARY KEY` mean the same on both drivers para/db ships, so the exact file runs verbatim on SQLite and PostgreSQL. There is no `BEGIN`/`COMMIT` in the file; para/db's migration runner owns the transaction.

> [!TIP]
> If your app already trusts para/db's CLI contribution (`[trust] commands = ["para/db"]`), the copied file also applies via `noeta migrate` like any other migration — it is plain SQL with the standard timestamp-prefix filename.

> [!NOTE]
> The store never garbage-collects expired rows — an expired row is simply invisible to `open` (every read filters `expires_at > now`). If table growth matters, sweep on your own schedule; the `expires_at` index makes it cheap:
>
> ```noeta ignore
> now = datetime.now().unix_ms() / 1000
> conn.execute("DELETE FROM sessions WHERE expires_at <= ?", [now])
> ```

## Revocation — deleting the row is the logout

`clear()` + `attach` is the user-facing revocation: the store DELETEs the row and overwrites the cookie with its expired form. The session's id deliberately survives `clear()` precisely so the store can still find the row to destroy. This is the logout the stateless store cannot give — the authoritative copy is server-side, so destroying it revokes the session for real; a copy of the id exfiltrated earlier no longer resolves to anything.

Because rows are authoritative, revocation also generalizes beyond a user clicking "log out". Any deleted row invalidates the id that named it, from anywhere that holds the `Connection`:

```noeta ignore
conn.execute("DELETE FROM sessions WHERE id = ?", [sid])   // revoke one known session
conn.execute("DELETE FROM sessions", [])                   // emergency: log out everyone
```

> [!NOTE]
> The `data` column is an opaque JSON blob keyed only by `id` — the store does not index sessions by user. A "log out this user everywhere" feature needs your app to record the user → session-id mapping itself.

## Swapping stores — the opt-in is one line

Handlers depend on `dyn SessionStore` and the `std.session` `Session`, never on `DbSessions` itself, so the choice of store is exactly the one `app.bind` line. Swap it for a `CookieSessions` and every handler above it is byte-for-byte unchanged — which also means an app can start stateless and move here only when it hits the cookie ceiling or needs revocation, without touching a route.

## Testing — session round-trips without a socket

A real `server.serve` blocks forever on a socket — never call it in a test — and aether's string-testable `dispatch_request` path threads no `Request`, so it cannot carry the cookies a session round-trip needs. The pattern (from [`examples/aether-sessions-db/`](examples/aether-sessions-db)) drives `app.serve_request` — aether's non-blocking dispatch core, the same function a deployed server mounts — with requests fabricated by an http client:

```noeta ignore
use std.http.client

http = client.new("http://app")

// Log in — the store mints an id, writes the row, sets the cookie.
r1 = http.prepare("POST", "http://app/login", "")
echo app.serve_request(r1).body()                      // "logged in"

// The minted id, read straight from the row (the cookie carries exactly this).
rows = conn.query("SELECT id FROM sessions", [])
sid = rows[0]["id"].as<string>() ?? ""

// Return with the cookie — the stored session rehydrates.
r2 = http.prepare("GET", "http://app/whoami", "", {"cookie": "sid=" ~ sid})
echo app.serve_request(r2).body()                      // "you are ada"
```

An in-memory database (`db.connect("sqlite::memory:")`) makes the whole thing hermetic, and querying the `sessions` table directly is the natural assertion that a login wrote a row and a logout deleted it.

## How it behaves

| Handler does | `attach` does |
|---|---|
| reads only (`open`, no `set`/`remove`/`clear`) | nothing — no row write, no `Set-Cookie` (a read is free) |
| writes (`set`/`remove`) | UPSERTs the row (data as JSON, fresh expiry) under its id — minted on the first write — and sets the id cookie |
| `clear()` | **DELETEs** the row and expires the cookie — a revocable logout |

`open` gives an empty session for a missing cookie, an unknown id, and an expired row alike — the one correct answer to "absent", "forged", and "expired", making them indistinguishable to a probe. The empty session is untagged, so its first write mints a *new* id rather than reusing a stale one.

The cookie is `HttpOnly`, `SameSite=Lax`, `Path=/`, and `Secure` per the `secure` argument. The id is 16 crypto-random bytes (128 bits), hex-encoded — an unguessable **key**, never a signed payload: the row is the source of truth, so the id carries no data and needs no verification, only unguessability.

Expiry is compared in-app against `datetime.now()` (a bound parameter), not in SQL, so the store's one statement runs unchanged on SQLite and PostgreSQL alike.

## Examples

- [`examples/aether-sessions-db/`](examples/aether-sessions-db) — the runnable end-to-end version: login, a stored round-trip, and a logout that deletes the row.
- [`examples/aether-db-demo/`](examples/aether-db-demo) — an aether app composing para/db storage: route-model binding and unit-of-work over a `para.db` repository.

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
