# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/aether_db** Noeta package (DB-backed sessions for para/aether; pure Noeta, composing para/aether + para/db), extracted from the noeta monorepo. Toolchain issues (the language, the `noeta` binary, `std.*`) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Versions that have to agree

- The package is at `0.3.0` and `noeta.toml` declares `toolchain = ">=0.5"`. CI installs whatever the **org-level** `NOETA_VERSION` variable names (`v0.6.0` today) — nothing in this repo pins the toolchain.
- The two dependencies bind under one `para` scope key as an array — the multi-package-per-scope form: `{ version = "^0.4", package = "para/aether" }` and `{ version = "^0.5", package = "para/db" }`. Both ranges admit the newest published sibling (para/aether 0.4.0, para/db 0.5.1), so neither is stale.
- **The declared `toolchain` floor understates the effective one.** para/db 0.5.1 — squarely inside this package's `^0.5` — itself declares `toolchain = ">=0.6"`, so an app composing the three needs noeta 0.6 whatever this manifest says. Raising a floor is a release decision with consumer consequences and belongs to a human; `toolchain-pin.yml` deliberately never touches it.
- Never bump the version or move a published `v*` tag as ordinary work — a release is its own commit plus a new tag.

## Build & test

- No cargo in *this* repo, but the examples pull para/db, whose native driver the composed toolchain cargo-builds — so both the `noeta` binary and a Rust toolchain must be on `PATH`, and the first run after a toolchain change is slow. Usually set nothing: `NOETA_TOOLCHAIN_SRC=<path to a noeta checkout>` only skips the git fetch, and `NOETA_TOOLCHAIN_REPO` matters only when composing against a fork or local clone.
- `noeta check <file>.noe` then `noeta test <file>.noe` from inside each `examples/*/` directory — that loop is the whole of `ci.yml`. `aether_db.noe` is compiled only *through* the examples; CI never checks the package root on its own.
- `ci.yml` has no fmt job here (some sibling repos have one), so run `noeta fmt --check .` yourself before committing.
- Both examples drive `app.serve_request`, aether's non-blocking dispatch core, rather than `server.serve` — so `noeta run` on them returns instead of blocking on a socket, and that is how you see their output.

## Gotchas

- **There is no `@test` anywhere in this repo**, so CI's `noeta test` step passes with nothing to run. The examples assert with top-level `echo` against `// expect:` headers, which only `noeta run` produces and nothing in this repo compares — `noeta check` is the only gate CI really applies. Verify a change by running the example and reading its output against its own `// expect:` block, and put new assertions in an `@test` block so CI gates them too.
- This repo is the **only** place para/aether and para/db are compiled together — neither dependency repo so much as mentions `aether_db`. A change to aether's `SessionStore`/`Session` currency or to para/db's `Connection` surface breaks here first, and here only.
- Methods are private by default (noeta 0.5+). Add `pub` only where the checker demands it — E0076 at a call site outside the type, E0015 on a trait impl — or where the README documents the method as API: `DbSessions.new` and the `SessionStore` methods `open`/`attach` are `pub`; `load`/`persist`/`revoke` stay private.
- `migrations/20260722120000_create_sessions.sql` is duplicated byte-for-byte into `examples/aether-sessions-db/migrations/` (the example's `conn.migrate("migrations")` reads its own copy, as a consumer would). A schema change has to move both copies and the README's column table.

## Conventions

- No lockfile is committed today: `examples/*/noeta.lock` are gitignored and regenerate on every run, and the package root — a library, resolved at the consumer — has none. The root lock is *not* gitignored, so if one is generated it is tracked, as in the sibling repos.
- Markdown never hard-wraps lines. **American English** throughout — code, comments, and docs (`behavior`, not `behaviour`).
- **Conventional commits** for every title. Commit each green slice as it completes, but **never `git push` without explicit authorization**.
- Implement in full — no stubs or TODOs; new functionality lands with tests. Keep `README.md` and this file true when layout or behavior changes.

## CI

`ci.yml` (a single `examples` job) installs the org-pinned `noeta` plus Rust 1.97.0 and checks + tests every example; `release.yml` reuses it as the gate, then `noeta publish`es a `v*` tag to the hosted registry with keyless Sigstore provenance via GitHub OIDC. Both are green — v0.3.0 published from this repo. `toolchain-pin.yml` is the fleet-identical file and scans `crates/*/` and `native/`, neither of which exists here: it rewrites nothing and builds nothing, so its green is **not** evidence this package survived a toolchain release — CI is. `docs-backfill.yml` re-uploads regenerated docs for an already-published tag.
