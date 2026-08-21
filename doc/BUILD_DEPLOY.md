# Sequoia-Configuration Build and Deployment

## Scope

This leaf is the Rust Cargo workspace for the on-device WebConfig daemon and
configuration engine. The Makefile is the single entry point for host builds,
tests, linting, vendoring, three release cross-targets, and development runs.
The release profile in `Cargo.toml` uses size optimization, LTO, one codegen
unit, aborting panics, and symbol stripping.

## 1. Prerequisites

- Rustup/Cargo with the pinned channel in `rust-toolchain.toml`, plus `rustfmt`
  and `clippy` components.
- Network access to Rust distribution services for `make toolchain`; the target
  also installs `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`, and
  `armv7-unknown-linux-gnueabihf` targets.
- A Unix host with `curl`; runtime `make stop` also needs `fuser`.

```sh
make toolchain              # one-time pinned toolchain and target setup
make help
```

## 2. Setup

## 3. Build

```sh
make build                  # debug workspace build
make release                # size-focused release workspace build
make cross                  # release build for all three configured targets
```

## 4. Test and lint

```sh
make test                   # all workspace targets; fake sysroot, no hardware
make lint                   # clippy -D warnings and rustfmt --check
make fmt                    # format the workspace in place
make check                  # build, test, and lint gate
```

## 5. Artifacts

```sh
make vendor                 # regenerate versioned offline Cargo vendor tree
make release-artifacts      # package source/vendor tarballs and print SHA-256
```

`target/` contains Cargo outputs. `release-artifacts` expects vendoring to have
been performed and produces the source/vendor artifacts consumed by the Yocto
recipe under `meta-webconfig`.

## 6. Deploy, install, and run

```sh
make run                    # real host sysroot; persistent state in ./data
WEBCONFIG_PORT=9443 make run
make run-mock               # temporary fake sysroot; no hardware side effects
WEBCONFIG_PORT=8443 make stop
```

`make run` reads live `/proc` and `/sys` and uses real systemd/update backends;
use it only on an appropriate development host. `make run-mock` is the safe
no-hardware validation path. There is no Makefile target that installs a
firmware image or deploys to a cloud/device.

## 7. Cleanup

```sh
make stop                   # stop the process on WEBCONFIG_PORT
make clean                  # cargo clean, removing target/
rm -rf data                 # optional: reset dev first-boot state
```

## 8. Verification caveats

Automated tests use a fake sysroot and do not validate systemd, live hardware
vitals, TLS on the target, or Yocto integration. The real `make run` path can
mutate host-facing state and binds the configured port. Device image creation
and flashing belong to Sequoia-OS; validate the daemon manually on the actual
Intel target after integration.