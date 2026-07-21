# mitch-lldap

<p align="center">
  <strong>A customized LDAP server for self-hosters — easy auth for your homelab.</strong><br/>
  <sub>Fork of <a href="https://github.com/lldap/lldap">lldap/lldap</a> • Adds branding, themes, multi-arch, and a fresh UI.</sub>
</p>

<p align="center">
  <a href="https://github.com/mitchelljfranklin/mitch-lldap/actions/workflows/rust.yml?query=branch%3Amitch-lldap">
    <img src="https://github.com/mitchelljfranklin/mitch-lldap/actions/workflows/rust.yml/badge.svg" alt="Build"/>
  </a>
  <a href="https://github.com/mitchelljfranklin/mitch-lldap/pkgs/container/mitch-lldap">
    <img src="https://img.shields.io/badge/container-ghcr-blue?logo=github" alt="Container"/>
  </a>
  <img src="https://img.shields.io/github/v/release/mitchelljfranklin/mitch-lldap?label=release" alt="Release"/>
  <img src="https://img.shields.io/badge/platform-amd64%20%7C%20arm64-blue?logo=raspberrypi" alt="Platforms"/>
  <img src="https://img.shields.io/badge/license-GPL--3.0-green" alt="License"/>
  <img src="https://img.shields.io/badge/MSRV-1.91.0-orange?logo=rust" alt="MSRV"/>
</p>

---

## Quick-start

```bash
docker pull ghcr.io/mitchelljfranklin/mitch-lldap:latest
# Continue to the full compose + env vars below
```

[Jump to install & configure](#install--configure)

---

## Why mitch-lldap?

| This fork adds | Built on upstream lldap |
|---|---|
| **Branding UI** — app name, logo, accent colors from the admin panel | **OPAQUE auth** — zero-knowledge password proof |
| **Dark mode** — system-aware, persisted, applied before first paint | **Multi-DB** — SQLite, PostgreSQL, MariaDB / MySQL |
| **Multi-arch** — amd64 + arm64 (Raspberry Pi 3B+, 4, 5) | **Simple LDAP** — no slapd configs, web UI for users & groups |
| **Bootstrap 5.3** — CSS tokens, variable-driven theming | **GraphQL API + CLI** — scriptable, integrates with everything |

Everything else — Terraform provider, OIDC bridges, PAM integration — stays in sync with upstream.

---

## Screenshots

<p align="center">
  <em>Screenshots coming soon — PR in progress</em>
</p>

<table>
  <tr>
    <td><img src="docs/screenshots/login.png" alt="Login page" width="100%"/><br/><sub>Login page with dark mode</sub></td>
    <td><img src="docs/screenshots/dashboard.png" alt="Dashboard" width="100%"/><br/><sub>Admin dashboard</sub></td>
    <td><img src="docs/screenshots/settings.png" alt="Settings" width="100%"/><br/><sub>Branding settings</sub></td>
  </tr>
</table>

---

## Install & configure

### Docker (recommended)

The image is published at `ghcr.io/mitchelljfranklin/mitch-lldap`. Available tags: `latest`, `v1`, `v1.x`, `v1.x.y`, and per-commit SHA.

```yaml
services:
  lldap:
    image: ghcr.io/mitchelljfranklin/mitch-lldap:latest
    restart: unless-stopped
    ports:
      - "17170:17170"   # Web UI
      # - "3890:3890"   # LDAP (only if services need direct LDAP access)
    volumes:
      - lldap_data:/data
    environment:
      - UID=10001
      - GID=10001
      - TZ=UTC
      # Generate secrets: ./generate_secrets.sh
      - LLDAP_JWT_SECRET=            # required — generate with ./generate_secrets.sh
      - LLDAP_KEY_SEED=              # optional — if set, delete any existing server_key* files
      - LLDAP_LDAP_BASE_DN=dc=example,dc=com
      - LLDAP_LDAP_USER_PASS=        # admin password — change after first login
      # Optional: database backends
      # - LLDAP_DATABASE_URL=postgres://user:pass@host/db
      # - LLDAP_DATABASE_URL=mysql://user:pass@host/db
      # Optional: SMTP for password reset
      # - LLDAP_SMTP_OPTIONS__ENABLE_PASSWORD_RESET=true
      # - LLDAP_SMTP_OPTIONS__SERVER=smtp.example.com
      # - LLDAP_SMTP_OPTIONS__PORT=587
      # - LLDAP_SMTP_OPTIONS__SMTP_ENCRYPTION=STARTTLS
      # - LLDAP_SMTP_OPTIONS__USER=no-reply@example.com
      # - LLDAP_SMTP_OPTIONS__PASSWORD=your-password

volumes:
  lldap_data:
```

### From source

```bash
cargo install wasm-pack
cargo build --release -p lldap
./app/build.sh                     # builds WASM frontend
cargo run -- run --config-file lldap_config.toml
```

For other platforms (Kubernetes, TrueNAS, distribution packages) see the [upstream install docs](https://github.com/lldap/lldap/blob/main/docs/install.md).

---

## Configuration reference

| Variable | Required | Default | Notes |
|---|---|---|---|
| `LLDAP_JWT_SECRET` | Yes | — | Generate with `./generate_secrets.sh` |
| `LLDAP_LDAP_USER_PASS` | Yes | — | Admin password on first run |
| `LLDAP_LDAP_BASE_DN` | No | `dc=example,dc=com` | Your LDAP base DN |
| `LLDAP_DATABASE_URL` | No | `sqlite://users.db?mode=rwc` | Postgres / MySQL / MariaDB URL |
| `LLDAP_HTTP_PORT` | No | `17170` | Web UI port |
| `LLDAP_LDAP_PORT` | No | `3890` | LDAP server port |
| `LLDAP_HTTP_URL` | No | — | Public URL (used in password reset emails) |
| `TZ` | No | `UTC` | Timezone for log timestamps |
| `LLDAP_VERBOSE` | No | `false` | Enable verbose logging |
| `LLDAP_KEY_SEED` | No | — | Encryption key seed |
| `LLDAP_SMTP_OPTIONS__ENABLE_PASSWORD_RESET` | No | `false` | Enable SMTP password reset |

All config values from `lldap_config.toml` can be overridden as `LLDAP_<SECTION>__<KEY>` env vars.

---

## Documentation

**This fork**
- [CHANGELOG.md](CHANGELOG.md) — version history and what changed
- [AGENTS.md](AGENTS.md) — development setup and contribution guide

**Upstream** (lldap/lldap)
- [Installation](https://github.com/lldap/lldap/blob/main/docs/install.md) — K8s, TrueNAS, package repos, cross-compilation
- [Usage](https://github.com/lldap/lldap#usage) — web UI, CLI, Terraform, GraphQL scripting
- [Client configuration](https://github.com/lldap/lldap#client-configuration) — compatible services, known configs, PAM integration
- [FAQ](https://github.com/lldap/lldap/blob/main/docs/faq.md) — login issues, DB migration, comparisons
- [Scripting](https://github.com/lldap/lldap/blob/main/docs/scripting.md) — GraphQL API examples
- [Example configs](https://github.com/lldap/lldap/tree/main/example_configs) — Authelia, Nextcloud, MinIO, PAM, and more

---

## Contributing

- **Fork features** — branch off `mitch-lldap`, open a PR targeting `mitch-lldap`
- **Upstream features** — branch off `main`, PR to [lldap/lldap](https://github.com/lldap/lldap)
- **Issues** — fork-specific bugs → this repo; general lldap issues → [upstream](https://github.com/lldap/lldap/issues)
- See [AGENTS.md](AGENTS.md) for local dev setup, build commands, and code style

---

## License & attribution

Built on [lldap/lldap](https://github.com/lldap/lldap) by [Valentin Tolmer](https://github.com/nitnelave) and contributors. Licensed under GPL-3.0.
