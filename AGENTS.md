# AGENTS.md — blaze

This file defines the architecture and contribution rules for this standalone Rust workspace. Run all build and verification commands from the repository root.

## Architecture

blaze is a **daemon-only** per-host sandbox orchestrator. All sandbox management is exposed via HTTP API; the binary only handles daemon lifecycle (start / reload / doctor).

Two-crate workspace:

- **blaze-core** (`crates/blaze-core`, library): policy engine, lifecycle state machine, backend selector, kernel hook registry, config schema. Zero I/O beyond local TOML/JSON parsing.
- **blazed** (`crates/blazed`, binary): daemon HTTP server (UDS + TCP), spawner implementations, metrics endpoint, CLI for daemon lifecycle commands.

Dependency direction: `blazed` → `blaze-core`. No reverse dependency.

## Build & Test

```bash
cargo build --workspace
cargo test --workspace
cargo clippy --workspace --all-targets
```

Platform: Linux (x86_64 + aarch64) for production. macOS builds succeed but spawners auto-downgrade to `MockSpawner`.

## Key Design Constraints

- **Daemon-only API model**: No CLI client for sandbox operations. All instance and template management is done via HTTP endpoints on UDS (`/run/blaze/api.sock`) or TCP (`:14159`). The CLI subcommands (`daemon start`, `daemon reload`, `daemon doctor`) only manage daemon lifecycle.
- **BackendSpawner trait**: All backend-specific process management is behind `BackendSpawner` (`spawn`, `probe`, `cleanup_orphan`, and defaulted `restore`/`restore_capability`) and `BackendInstance` (`backend`, `try_wait`, `kill`, plus defaulted `pause`, `resume`, `snapshot`, and the capture-orchestration hooks `quiesce_for_capture`/`unquiesce_after_capture`, which delegate to pause/resume and are overridden as no-ops by backends whose capture primitive freezes the workload itself; the quiesce must hold until `unquiesce_after_capture`, because storage synchronization and rootfs capture run after `snapshot` returns). Adding a new backend means implementing the required methods and registering it in `daemon::build_spawners()`.
- **Policy-driven backend selection**: Workload class → policy file → prioritized backend list. The daemon probes backends at startup and selects the first available. Never hardcode backend preference in application logic.
- **Lifecycle state machine**: 13 states. The main branches are Pending →
  Creating → Running, Running ↔ Paused → Checkpointed, and
  Running → Restoring → Running for checkpoint restore. Hibernation follows
  Running → Hibernating → Hibernated → Resuming → Running; compensation can
  return Hibernating to Running or Resuming to Hibernated. Any non-terminal
  state can enter Destroyed; incomplete cleanup enters RecoveryRequired. State transitions are enforced by
  `blaze_core::lifecycle`. Do not bypass via direct field mutation.
- **MockSpawner fallback**: When the configured backend binary is missing or fails `probe()`, the daemon auto-downgrades to `MockSpawner` with a warning. This keeps API/integration tests functional without a real backend.

## Adding a New Backend

1. Add a variant to `BackendKind` in `crates/blaze-core/src/backend.rs`
2. Implement `BackendSpawner` in `crates/blazed/src/spawner.rs`
3. Register the new spawner in `daemon::build_spawners()`
4. Add a corresponding `[backends.<name>]` section in config schema (`crates/blaze-core/src/config.rs`)
5. Add policy support: allow the new backend kind in policy `backends` priority lists
6. Add unit tests for `probe()` and `spawn()` (use mock paths for CI)

## Configuration

Runtime config: `/etc/blaze/config.toml` + `/etc/blaze/policies/*.toml`

Development config: `examples/config.toml` + `examples/policies/`

When modifying config schema, update both the Rust struct in `config.rs` and the example files.

Packaging assets live in `dist/`: `blaze.spec`, `blazed.service`, and `tmpfiles-blaze.conf`. Keep package installation paths and service settings aligned with runtime defaults.

## Rust Conventions

### Comments and Documentation

- Write code and comments in English. Explain intent, invariants, preconditions, side effects, and protocol contracts rather than restating names or signatures.
- Use `//!` for concise module documentation, `///` for public items (including significant fields and variants), and `//` to explain non-obvious implementation choices. Avoid redundant documentation on private helpers.
- Begin rustdoc with a standalone summary. Add `# Errors`, `# Panics`, `# Safety`, and runnable `# Examples` when applicable; unsafe functions must document caller obligations.
- Do not leave bare TODOs without an owner and context, commented-out code, or author, timestamp, changelog, or issue-tracking notes in comments. Use version control and PR descriptions for history.

### Module Layout and Dependencies

- Use parent `.rs` modules and matching child directories; do not create `mod.rs`, except `tests/common/mod.rs` for shared integration-test helpers.
- Declare third-party versions and shared features in `[workspace.dependencies]`; reference them from member crates with `workspace = true`. Member crates may add features when needed.
- Check existing dependencies before adding an equivalent crate. Discuss major-version dependency upgrades before making them.

### Error Handling

- Library crates own named error enums derived with `thiserror`, wrapping upstream errors with `#[from]` where appropriate. Binaries may use `anyhow::Result`.
- Avoid `unwrap()`, `expect()`, and `panic!()` in library implementation code unless a comment establishes why failure is unreachable; prefer encoding guarantees in types.
- Prefer `?` for propagation. Error messages must include useful failure context and relevant values.
- Keep Clippy allowances narrowly scoped and explain their purpose. Never remove tests or assertions to make checks pass.

## Documentation Conventions

- Treat source code as the authority; verify CLI, HTTP API, and configuration examples against their implementations. Do not describe planned features as available.
- Keep English and Chinese documentation semantically equivalent, with identical command examples. Use `FILE.md` / `FILE_zh.md` for adjacent pairs and `en/` / `zh/` for user guides. Add reciprocal language links to paired files. Keep `AGENTS.md` in English.
- Keep the README as the project entry point with positioning, installation, and basic usage. Store architecture and protocol design documents in `docs/design/`; place expanded user and developer guides under `docs/user-guide/` and `docs/developer-guide/` when needed.
- Update the README and relevant reference or design documents alongside CLI, API, configuration, or architecture changes. Changelog entries describe user-visible effects and belong in release version-bump changes; update both language versions together.

## Commit Conventions

Use `type: imperative description` for changes in this repository. Write the subject in English, start the description with a lowercase letter, omit the trailing period, and keep the subject within 50 characters. Mark breaking changes with `!` before the colon. Examples:

```
feat: add snapshot backend
fix: handle missing rootfs gracefully
```

Keep each commit focused on one logical change and independently buildable. For non-trivial changes, explain the architectural choice, rationale, and limitations in the body. Amend or fix up issues introduced by the same branch rather than creating separate fix commits.

Include `Signed-off-by: Name <email>` and, for AI-assisted commits, an `Assisted-by: <tool>:<version>` trailer above it. Use the actual tool version; do not invent a version or use `latest`.

Version bumps use `chore: bump version to X.Y.Z`, update all version-bearing files and both changelogs together, and are the last commit in a feature branch.

## Verification

Before committing:

```bash
cargo fmt --all -- --check
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo doc --workspace --no-deps   # ensure no broken intra-doc links
```
