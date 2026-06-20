# Security and Capability Model

## Security boundary

The generated wasm module should not acquire ambient host power. Standalone WASI output and Node-host-shim output are separate target profiles. Host powers must be visible in runtime link plans and capability manifests.

## Rules

- Every Node host import starts with `host.`.
- Every host import has a capability reason.
- Standalone manifests cannot require node host imports.
- `--host-deny` rejects host-dependent programs.
- Filesystem, env, random, clock, stdin/stdout/stderr capabilities require explicit reasons when enabled.

## Threats this model addresses

| Threat | Control |
|---|---|
| Accidental Node API use in standalone builds | `--host-deny`, manifest validation |
| Hidden runtime dependency | runtime catalog dependencies and link-plan tests |
| Unreviewed filesystem/env/clock/random access | capability reasons |
| Target drift | canonical `ExecutionTarget` and ABI metadata |

## What this does not yet guarantee

- Full sandbox proof for every host shim implementation.
- Browser/component-model isolation.
- Complete ECMAScript security semantics for every builtin.

Those require future target-specific design docs and tests.
