# AGENTS.md

Fork of [lldap/lldap](https://github.com/lldap/lldap) — Light LDAP server with a WASM frontend.

## Repo identity

- **Fork owner**: mitchelljfranklin
- **Fork branch**: `mitch-lldap` (default, protected — requires PRs)
- **Upstream**: `lldap/lldap` (`upstream` remote, `main` branch)
- **Container**: `ghcr.io/mitchelljfranklin/mitch-lldap`
- **Dual README**: `.github/README.md` (fork — displayed on repo) / `README.md` (root — upstream, avoid editing)

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

## Gotchas

- **`default-members = ["server"]`** — bare `cargo build` only builds the server, not the full workspace.
- **Integration tests are Unix-only** (`#[cfg(unix)]` on all three test binaries). They start LLDAP as a child process using nix signals. Will compile but produce 0 tests on Windows.
- **Indexmap pinned** in `app/Cargo.toml`: `indexmap = "=1.6.2"` — needed for aHash WASM compatibility. Do not bump.
- **`yew_form` from git**, not crates.io — pinned to a specific rev: `jfbilodeau/yew_form@4b9fabf`.
- **`key_seed`**: if set in config, delete any existing `server_key*` files to avoid startup errors.
- **MSRV**: 1.91.0 (both CI and Cargo.toml `rust-version`).
- **Codecov job in CI** only runs on `lldap/lldap` — skipped on forks.
- **Database schema** auto-migrates at startup (SeaORM). No manual migration needed.

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
