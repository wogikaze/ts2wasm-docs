# W1 (Standalone WASI execution) — remaining work

Last updated: 2026-05-08

## Gate A precondition

Gate A requires `cargo nextest run` to pass. W1 is a foundation workstream alongside W0.

## Current status — implemented

- **console.log**: Works via WASI `fd_write`. Tested via `m_standalone_wasi.rs` and `m11_host_deny.rs`.
- **stdin support**: Works via WASI `fd_read`. Tested via stdin fixture tests in `m2_node_diff.rs` (bun_stdin, m6_stdin, assert_stdin_fixture).
- **WASI import resolution**: Core WASI snapshot_preview1 imports resolved in emitter.
- **WASI filesystem**: Read/write working via host-deny tests.
- **WASI clock**: `clock_time_get` for Date.now() via host-deny tests.
- **WASI random**: `random_get` for Math.random() via host-deny tests.
- **Standalone purity**: `m11_host_deny.rs` verifies standalone fixtures pass without Node host imports.

## Remaining work

### Test coverage

- [ ] **Improve standalone test coverage**: `m1_iwasm.rs` and `m_standalone_wasi.rs` should exercise more output scenarios.
- [ ] **Stdin edge cases**: Test empty stdin, large stdin, pipe stdin.

### WASI surface

- [ ] **WASI args**: `args_get`/`args_sizes_get` — not yet wired for process.argv.
- [ ] **WASI env**: `environ_get`/`environ_sizes_get` — not yet wired for process.env.
- [ ] **WASI clock resolution**: `clock_res_get` not imported — may be needed for performance measurement.
- [ ] **WASI proc_exit**: Currently traps; should exit cleanly with exit code.

### Host-deny boundary

- [ ] **Expand host-deny test matrix**: Current `m11_host_deny.rs` has 12 function tests. Should cover more host import scenarios.
- [ ] **Manifest audit**: Verify `--emit-manifest` behavior for all WASI capabilities used.

## Reference

- Gate definitions: `docs/11-shared-definitions.md`
- Test files: `crates/cli/tests/m1_iwasm.rs`, `m_standalone_wasi.rs`, `m11_host_deny.rs`
- WASI support: WASI Preview 1 via `wasi_snapshot_preview1` imports
