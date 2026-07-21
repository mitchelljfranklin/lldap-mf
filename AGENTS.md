# AGENTS.md

Fork of [lldap/lldap](https://github.com/lldap/lldap) — Light LDAP server with a WASM frontend.

## Repo identity

- **Fork owner**: mitchelljfranklin
- **Fork branch**: `mitch-lldap` (default, protected — requires PRs)
- **Upstream**: `lldap/lldap` (`upstream` remote, `main` branch)
- **Container**: `ghcr.io/mitchelljfranklin/mitch-lldap`
- **Dual README**: `.github/README.md` (fork — displayed on repo) / `README.md` (root — upstream, avoid editing)

## Branch structure

- **`main`** — tracks `upstream/main`. Fork-only changes should not land here.
- **`mitch-lldap`** — fork's deployment branch (default, protected — requires PRs).
- All new features intended for upstream: branch off `main`.
- All fork-only features: branch off `mitch-lldap`.

## Upstream sync routine

```bash
git checkout main
git pull upstream main
git push origin main
git checkout mitch-lldap
git merge main  # resolve conflicts, keep both upstream + fork changes
cargo build --workspace  # verify merge
```

## Build & verify

```sh
cargo build --workspace          # ~3m, timeout 300s
cargo test --workspace --lib     # unit tests (integration tests are Unix-only)
./app/build.sh                   # WASM frontend, ~3m, timeout 300s
cargo fmt --all --check          # <5s
cargo clippy --tests --all -- -D warnings  # ~90s, timeout 120s
```

**If you change GraphQL types** you must regenerate the schema or CI fails:
```sh
./export_schema.sh               # ~80s, timeout 120s
```
This regenerates `schema.graphql` (checked into git). The `rust.yml` workflow diffs it against a fresh export and fails if stale.

## Gotchas

- **`default-members = ["server"]`** — bare `cargo build` only builds the server, not the full workspace.
- **Integration tests are Unix-only** (`#[cfg(unix)]` on all three test binaries). They start LLDAP as a child process using nix signals. Will compile but produce 0 tests on Windows.
- **Indexmap pinned** in `app/Cargo.toml`: `indexmap = "=1.6.2"` — needed for aHash WASM compatibility. Do not bump.
- **`yew_form` from git**, not crates.io — pinned to a specific rev: `jfbilodeau/yew_form@4b9fabf`.
- **`key_seed`**: if set in config, delete any existing `server_key*` files to avoid startup errors.
- **MSRV**: 1.91.0 (both CI and Cargo.toml `rust-version`).
- **Codecov job in CI** only runs on `lldap/lldap` — skipped on forks.
- **Database schema** auto-migrates at startup (SeaORM). No manual migration needed.
- **Multi-DB**: SQLite (default), PostgreSQL, and MariaDB/MySQL are all supported. Any DB schema or query changes must work across all three backends. SeaORM abstracts most differences, but raw SQL, migration scripts, and conflict handlers must be backend-compatible. Test migrations against all backends when possible.
- **`Cargo.lock` on Windows** may resolve differently than Linux CI. If `cargo build` fails unexpectedly on Windows, run `cargo update` to refresh the lockfile.

## Architecture

```
server/                → lldap binary (Actix-web + LDAP + GraphQL)
app/                   → lldap_app WASM frontend (Yew, cdylib)
migration-tool/        → lldap_migration_tool (OpenLDAP migration CLI)
set-password/          → lldap_set_password CLI
crates/auth            → OPAQUE protocol
crates/ldap            → LDAP server implementation
crates/graphql-server  → Juniper GraphQL
crates/sql-backend-handler → SeaORM DB layer
crates/frontend-options    → shared Options struct (password_reset_enabled)
```

Default ports: LDAP 3890, HTTP 17170, LDAPS 6360 (disabled).

## Fork vs upstream

The `mitch-lldap` branch adds:
- Branding/admin settings UI (app name, logo, colors, theme)
- Bootstrap 5.3.3 upgrade + dark mode + CSS design tokens
- Multi-arch Docker build (amd64, arm64 via cross-compilation)
- Fork-specific CI, docs, footer, and repo identity

The `main` branch tracks upstream and has a stripped-down Bootstrap upgrade ready for PR.

## Docker

- Root `Dockerfile` — standalone multi-stage build, compiles everything from source
- `.github/workflows/Dockerfile.ci.multiarch` — CI image, pre-built binaries via `cross`
- Fork workflow triggers on **release published** or **manual dispatch** only
- For local Docker dev: `docker run -e LLDAP_JWT_SECRET=secret -e LLDAP_LDAP_USER_PASS=pass -p 17170:17170 -v lldap_data:/data mitch-lldap:test`

## CI

- **`rust.yml`** — build, test, clippy, fmt, and GraphQL schema check. Triggers on `main` and `mitch-lldap` branches.
- **`docker-build-fork.yml`** — container publish to `ghcr.io/mitchelljfranklin/mitch-lldap`. Triggers on release published or manual dispatch.

## Quality gates

Before considering work done:

- **No stubs.** No `TODO`/`FIXME`, no empty/throwing function bodies, no dead or commented-out code. Intentional empty states must be clearly labelled.
- **Lint + build clean.** `cargo fmt --all --check`, `cargo clippy --tests --all -- -D warnings`, and `cargo build --workspace` must pass. For frontend changes, `./app/build.sh` must also pass.
- **Types are explicit.** No `unwrap()` or `expect()` without justification. Validate external input. Use `thiserror` for structured errors.
- **Tests pass.** `cargo test --workspace --lib` must pass. Integration tests (Unix-only) pass on CI.

## Code style

- **No comments unless explicitly requested.** Clarity comes from names and structure, not comments. When a comment is requested, explain *why*, not *what*.
- **Full descriptive names.** `calculate_display_name`, not `calc_nm`. Variables are noun phrases, functions are verb phrases. Single-letter variables only in closure parameters (`|x|`) and loop indices.
- **One thought per line.** Break chained operations into intermediate variables with descriptive names. No dense one-liners.
- **Early returns over deep nesting.** Use guard clauses and `let-else`. Functions should read top-to-bottom.
- **No over-engineered abstractions.** Don't create a trait for a two-method type used once. Every abstraction must reduce total cognitive load.
- **Duplicate code is noise.** If logic appears in two places, extract it into a shared location.
- **Match existing conventions.** When editing a file, mimic its import style, error handling pattern, and naming. Don't introduce a different pattern in the same module.
- **Human-readable output.** Generated code should look like a human wrote it — clean formatting, logical grouping, descriptive names, and no mechanical boilerplate patterns. If it looks generated, it needs more polish.
- **Multi-DB awareness.** Any change touching database queries, migrations, or schema must compile and pass tests across SQLite, PostgreSQL, and MariaDB/MySQL. Avoid backend-specific SQL or SeaORM API calls that don't work uniformly.
