# Sequoia-Configuration

## Project Summary

Sequoia-Configuration is an on-device, browser-based configuration system for the Sequoia
Yocto/embedded-Linux platform (Intel x86 RT 5). One small Rust daemon serves a vanilla-JS frontend
over TLS; board-specific data lives in a single board-manifest file
(`board-manifests/sequoia.json`).

> The full design specification lives in [`docs/webconfig-spec.md`](docs/webconfig-spec.md).

The repository is a Rust workspace with an engine crate, an Axum daemon, a vanilla-JavaScript
frontend, JSON schemas, board manifests, a Yocto layer, and documentation. The daemon uses a
single-thread Tokio runtime and rustls rather than OpenSSL. Configuration state is represented by
JSON and ring files rather than a database. The frontend is schema-driven: schemas describe settings
and validation, while shared page code renders the forms. The documented API surface includes
authenticated configuration sections, CSRF protection, role checks, SSE, and system interactions
through RAUC/zbus adapters. A fake-sysroot harness supplies mocks for tests without requiring a
physical board.

Networking changes have a provisional commit/confirm workflow so an operator can reconnect and
confirm a change before the rollback window expires. The repository also carries generic and Sequoia
board manifests, release and offline-vendor settings, and a `meta-webconfig` Yocto integration. The
checked-in README describes reproducible x86_64, aarch64, and armv7 targets and the root Makefile
commands. Those declared capabilities are documented as implemented only where the
source/configuration inventory supports them; production TLS, real RAUC/zbus behavior, and board
deployment still need manual validation.

## Feature Inventory

| # | Name / Description | Implemented? | Test Coverage? | Manual Test |
| --- | --- | --- | --- | --- |
|   1 | Rust workspace and reproducible release profile | Implemented  | Build/check command documented                | [ ] Manual verification required |
|   2 | Configuration engine and JSON/ring-file state   | Implemented  | Partial; engine test module present           | [ ] Manual verification required |
|   3 | Axum HTTP daemon                                | Implemented  | Partial; `crates/webconfigd/src/tests.rs`     | [ ] Manual verification required |
|   4 | TLS serving with rustls                         | Implemented  | Unknown for real certificates                 | [ ] Manual verification required |
|   5 | Authentication, roles, and CSRF protection      | Implemented  | Partial; daemon tests                         | [ ] Manual verification required |
|   6 | Schema-driven frontend forms and validation     | Implemented  | Unknown; no dedicated frontend test inventory | [ ] Manual verification required |
|   7 | Network commit/confirm rollback workflow        | Implemented  | Unknown from focused tests                    | [ ] Manual verification required |
|   8 | SSE monitoring/event endpoints                  | Implemented  | Unknown from focused tests                    | [ ] Manual verification required |
|   9 | RAUC and zbus system adapters                   | Partial      | Mock-oriented coverage only                   | [ ] Manual verification required |
|  10 | Sequoia and generic board manifests             | Implemented  | Schema/config validation required             | [ ] Manual verification required |
|  11 | Yocto `meta-webconfig` packaging layer          | Implemented  | Not found; image build required               | [ ] Manual verification required |
|  12 | Offline vendoring and cross-target builds       | Implemented  | Command-level evidence only                   | [ ] Manual verification required |

## Build and Test

A field engineer or operator points a browser at the device (`https://<device>:8443`) and configures
it: system identity, networking, services, updates, monitoring, logs, alerts, security, and I/O —
all behind authenticated, role-based access with a commit/confirm rollback safety net for networking
changes.

- **One daemon**, `current_thread` tokio runtime, rustls (no OpenSSL), no database (JSON + ring
  files). Target: <15 MB RAM, 3-6 MB stripped binary.
- **Vanilla JS frontend**, no build step, no node at build time, <500 KB total.
- **Schema-driven rendering**: adding a setting is a schema file — zero UI code.
- **Flagship networking rollback**: network changes are provisional; reconnect and confirm within a
  window or they auto-roll-back.
- **Real + mock system interactions**: RAUC/zbus talk to real backends on-device and to mocks on the
  fake-sysroot test harness (no hardware to test).

## Repository layout

```
Cargo.toml                  # workspace, pinned deps, exact release profile
crates/
  webconfig-engine/         # config store, appliers, schema, manifest, alert/monitoring
  webconfigd/               # axum HTTP daemon (TLS, auth, CSRF, 9 section APIs, SSE)
frontend/                   # vanilla JS: index.html, styles.css, schema-form.js, pages.js, app.js
schemas/                    # *.schema.json — schema-driven rendering + validation
board-manifests/            # sequoia.json (board-specific data) + generic.yaml fallback
meta-webconfig/             # Yocto layer: daemon, static assets, schemas, manifest, hardened unit
docs/                       # spec, API reference, schema format, board-manifest spec, planning/audits
Makefile                    # build | release | test | lint | fmt | vendor | cross | run | clean
```

## Build & test

The repo-root Makefile is the single entry point (per SoCcentric conventions):

```bash
make build    # debug build (host target)
make release  # exact spec release profile (opt-level=z, lto, panic=abort, stripped)
make test     # engine + HTTP integration tests (fake sysroot, no hardware)
make lint     # clippy -D warnings + rustfmt --check
make vendor   # regenerate the offline crate tarball for reproducible builds
make cross    # build all three cross-targets: x86_64, aarch64, armv7
make run      # run the daemon against a tempdir fake sysroot (dev)
```

Validation gate (used by the git workflows and CI): `make build && make test && make lint`.

## Cross-targets

`x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`, `armv7-unknown-linux-gnueabihf`. Vendored
crates + `CARGO_NET_OFFLINE=1` make builds fully offline and reproducible.

## Deploying (Yocto)

Add the layer and packages to your image:

```bitbake
IMAGE_INSTALL:append = " webconfigd webconfig-static webconfig-schemas webconfig-manifest"
```

Map your BSP `MACHINE` to its board manifest in
`meta-webconfig/recipes-core/webconfig/webconfig-manifest.bbappend`. See
[`meta-webconfig/README.md`](meta-webconfig/README.md).

## Documentation

- [Design specification](docs/webconfig-spec.md)
- [REST API reference](docs/api-reference.md)
- [Schema format](docs/schema-format.md)
- [Board-manifest format](docs/board-manifest.md)
- [Production-readiness audit](docs/planning/MasterPlan.md) and its
  [acceptance checklist](docs/planning/Checkpoint.md)

Use `make build && make test && make lint` for the documented validation gate.

## License

MIT. See [LICENSE](LICENSE).
