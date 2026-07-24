# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/aether_db** Noeta package (DB-backed sessions for para/aether; pure Noeta, composing para/aether + para/db), extracted from the noeta monorepo. Toolchain issues (the language, the `noeta` binary, `std.*`) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Repo layout

- `noeta.toml` — the package manifest (`name = "para/aether_db"`). Pure Noeta (no `native` key), but it **depends on two sibling packages** — para/aether and para/db — bound under one `para` scope key as an array (multi-package-per-scope resolution). Those deps are currently the pre-publish `git = "file:///home/niklas/Code/para/para-*"` form; they flip to github URLs / registry versions at publish.
- `sessions.noe` — the whole surface: the `DbSessions` class implementing para/aether's `SessionStore` over a para/db `Connection`.
- `migrations/` — the one `sessions`-table migration consumers copy or point `conn.migrate` at.
- `examples/*/` — each a standalone package depending on this repo via `{ path = "../.." }` alongside the sibling git deps.
- `.github/workflows/` — CI (`ci.yml`) and the tag-triggered registry publish (`release.yml`).

## Build & test

No cargo in *this* repo, but the examples pull para/db, whose native driver the composed toolchain cargo-builds — so running them needs both the `noeta` binary and a Rust toolchain on `PATH`.

- `noeta check <file>.noe` / `noeta test <file>.noe` in each `examples/*` directory is the test suite.
- The composed toolchain must resolve the same toolchain repo the native crates' Cargo.toml declares. Pre-publish that means `NOETA_TOOLCHAIN_REPO=file:///home/niklas/Code/lang` (a mismatch links two copies of the extension ABI — every impl fails with a two-`Extension`-traits E0308); optionally `NOETA_TOOLCHAIN_SRC=<path to a noeta checkout>` to skip the git fetch.

## Conventions

- `noeta.lock` files under `examples/` **are committed** — leave resolved locks in place; don't delete or regenerate them gratuitously.
- Markdown never hard-wraps lines; American English throughout.
- Conventional commits. Never move a published `v*` tag — a release is a new tag.

## CI

`ci.yml` checks and tests every example with a pinned released `noeta` (plus the pinned Rust toolchain, for para/db's native half); `release.yml` publishes the tag to the hosted registry (`noeta publish`, keyless Sigstore provenance via GitHub OIDC). Both go green only once the toolchain repo is published under github.com/noeta-lang/noeta and the `file:///` deps are flipped.
