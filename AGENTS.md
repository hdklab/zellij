# AGENTS.md

Zellij — a terminal workspace/multiplexer in Rust. A Cargo workspace whose own build system is
`cargo xtask` (a crate in `xtask/`, aliased in `.cargo/config.toml`). The `zellij` binary hosts a
client (`zellij-client`), a server (`zellij-server`: screen, tabs, panes, pty, plugin runtime) and
shared types/config/KDL parsing (`zellij-utils`). Builtin UI plugins are separate crates under
`default-plugins/` compiled to `wasm32-wasip1` and embedded into the binary.

## Dev environment

- Rust toolchain is pinned by `rust-toolchain.toml` (currently 1.95.0, includes `rustfmt`, `clippy`,
  and the `wasm32-wasip1` + `x86_64-unknown-linux-musl` targets). MSRV is whatever that file says.
- `protoc` is required for normal builds (compiles the `.proto` files behind the plugin/IPC API).
- Optional: `cargo-nextest` (used by `cargo xtask integration-test`), `docker`/`podman` + `docker compose`
  (for the SSH e2e tests), `cargo-zigbuild` + `zig` (only for the macOS e2e path).

## Build & test

Always prefer the xtask wrappers; they build plugins and the binary together.

```sh
cargo xtask                 # format, build, test (the "make" pipeline)
cargo xtask format          # cargo fmt across every workspace member; --check for CI mode
cargo xtask build           # plugins + binary; -r for release, --plugins-only, --no-plugins, --no-web
cargo xtask test            # builds plugins, then cargo test per crate; --no-web / --no-window
cargo xtask run -- -l strider   # run the debug build; args after -- go to zellij
cargo q                     # alias: re-run without rebuilding plugins (--quick-run)
cargo xtask integration-test [--no-opt] [--serial] [-- <nextest filter>]
cargo xtask proto           # regenerate zellij-utils/assets/prost* (needs protoc)
cargo xtask assets          # regenerate the web-client bundle; --check verifies it
cargo xtask ci e2e --build  # then: cargo xtask ci e2e --test (needs docker compose up -d)
cargo test -p xtask         # tests for the build system itself
```

CI (`.github/workflows/rust.yml`, `e2e.yml`) runs exactly: `cargo xtask build`, `cargo xtask test`,
`cargo xtask format --check`, `cargo xtask test --no-web`, `cargo xtask integration-test`,
`cargo test -p xtask`, `cargo xtask assets --check`. On Windows it runs per-crate
`cargo test --no-default-features -p <crate> --lib` instead of `cargo xtask test`.

## Conventions

- Error handling: `use zellij_utils::errors::prelude::*;`, return `anyhow::Result`, attach
  `.context("...")` at each fallible step, log recoverable failures with `.non_fatal()` (not
  `log::error!`), and add a `ZellijError` variant for errors callers must inspect. See
  `docs/ERROR_HANDLING.md`. New `unwrap()` calls are discouraged.
- Logging goes through the `log` crate; runtime log is `$TMPDIR/zellij-<uid>/zellij-log/zellij.log`
  (truncated at 100KB). `zellij --debug` also dumps raw pty bytes per pane as `zellij-<pane_id>.log`.
- Plugins: one crate per plugin under `default-plugins/<name>/`, each with
  `.cargo/config.toml` pinning `target = "wasm32-wasip1"`; must be listed in both
  `workspace.members` (`Cargo.toml`) and `workspace_members()` (`xtask/src/main.rs`).
- Formatting: `rustfmt` with only `match_block_trailing_comma = true` (`.rustfmt.toml`); 4 spaces,
  LF, final newline (`.editorconfig`); YAML uses 2 spaces.
- Commits: Conventional Commits with a scope, e.g. `fix(sessions): ...`, `feat(window): ...`;
  released changes also get a `CHANGELOG.md` entry.
- Deep dives live in `docs/ARCHITECTURE.md` (screen/pane/boundary/pty model) and
  `docs/TERMINOLOGY.md` (ANSI/VT, CSI, OSC, pty) — read them before touching rendering code.

## Pitfalls

- Do not set `CARGO_TARGET_DIR` or change `build.target-dir` in `.cargo/config.toml`: debug builds
  embed plugins from the hard-coded `<repo>/target/wasm32-wasip1/debug` path at compile time. Use a
  symlinked `target/` directory if you want to share artifacts. Release builds embed from
  `zellij-utils/assets/plugins` instead and are unaffected.
- Editing a plugin is not enough — rebuild with `cargo xtask build` (or `cargo xtask test`), otherwise
  the binary still embeds the old `.wasm`.
- Generated, do not hand-edit: `zellij-client/assets/app.js` (bundled from the sibling module
  `*.js` files by `cargo xtask assets`), `integrity.json`, the rewritten `index.html`, and
  `zellij-utils/assets/prost*` (`cargo xtask proto`). `zellij-utils/assets/plugins/*.wasm` are
  checked in but are build outputs.
- `cargo xtask integration-test` runs the in-process harness in `zellij-integration-tests/tests/`;
  it uses the `dev-opt` profile (slow one-time dep build, fast tests; `--no-opt` is ~7x slower) and
  `.config/nextest.toml` sets a 30s slow-test timeout with 4 threads required for `nested_*` tests.
- Snapshot tests use `insta`; snapshots are checked-in `.snap` files and failed comparisons leave
  gitignored `*.snap.new` files. `zellij-integration-tests/preview_snapshots.sh <test> [file]` opens
  a snapshot in vim in a new pane.
- SSH e2e tests bind host port `127.0.0.1:2222` and mount `./target` into the container; the binary
  must be rebuilt (`cargo xtask ci e2e --build`) before re-running tests.
- The web server and the native window are optional features (`--no-web`, `--no-window`); code behind
  them must keep both no-feature builds compiling.
