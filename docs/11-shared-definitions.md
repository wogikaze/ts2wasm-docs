# Shared Definitions

This document holds cross-cutting terms and schemas referenced by the rest of `docs/`.
Do not redefine these tables in feature docs; link here and add only local context.

## Core Terms

| Term | Definition |
|---|---|
| Source | Input JavaScript/TypeScript text plus path/span information. |
| AST | Syntax tree from `syntax`/`frontend`; owns syntax shape, not runtime semantics. |
| Builtin-resolved AST | AST-like representation where names and builtin identities are classified. |
| HIR | Semantic IR for JavaScript operations and completion behavior. |
| MIR | Lower-level validated IR for backend emission. |
| LoweredProgram | Legacy backend-facing IR still used by the default build path. |
| RuntimeFn | Cataloged runtime helper with declared dependencies, imports, capabilities, strings, globals, and signatures. |
| RuntimeLinkPlan | Validated transitive runtime closure required by a program. |
| CapabilityManifest | Auditable JSON description of host/WASI powers needed by output. |
| Target | Execution ABI/profile selected by `ExecutionTarget`. |

## Diagnostic Terms

- `SourceDiagnostic`: source-originating diagnostic with mandatory span.
- `InternalDiagnostic`: compiler invariant/backend/internal diagnostic with optional span.
- `Diagnostic`: legacy shared diagnostic with code, message, optional span, and optional phase.
- `DiagCode`: stable error classification used by CLI, server, tests, and coverage.

## Test Status Schema

All test records use one of these statuses. A plain skip is not a valid status.

| Status | Meaning | Required detail |
|---|---|---|
| `pass` | Behavior matched the expected oracle. | suite, case, target |
| `fail` | Implementation bug or semantic mismatch. | expected, actual, reproduction target |
| `unsupported` | Compiler rejected an unsupported feature. | reason, feature label, tracking id |
| `blocked` | External runtime, host, or toolchain blocker. | blocking condition, owner or upstream |
| `skip-with-reason` | Explicit environment/configuration exclusion. | reason, condition, recheck rule |

`build_pass` means the compiler produced a wasm artifact. It does not imply semantic correctness unless `semantic_checked=true` or separate differential evidence is present.

## Capability Manifest Schema

`CapabilityManifest` is the auditable boundary between generated wasm and host power.
The current schema version is `1`; the named constant lives in `crates/shared/src/capability.rs`.

```json
{
  "schema_version": 1,
  "target": "wasm32-wasi-p1",
  "target_id": "wasm32-wasi-p1",
  "target_aliases": ["wasm32-wasi"],
  "runtime_abi_name": "ts2wasm-runtime-abi",
  "runtime_abi_version": 2,
  "standalone": true,
  "wasi": {
    "stdin": false,
    "stdout": true,
    "stderr": false,
    "args": false,
    "env": false,
    "clock": { "realtime": false },
    "filesystem": { "read": [], "write": [], "preopens": [] },
    "random": false
  },
  "node_host": {
    "required": false,
    "imports": []
  },
  "capability_reasons": {
    "wasi.stdout": ["console.log"]
  }
}
```

Manifest rules:

- `standalone=true` cannot coexist with `node_host.required=true`.
- Every enabled WASI capability must have a `capability_reasons` entry.
- Every Node host import must start with `host.` and have a reason.
- New fields must be optional or defaultable. Removing or renaming a field is a schema-breaking change and requires a version bump plus fixture/dashboard updates.

## Optimization And Safety Modes

Optimization level and semantic safety mode are separate concepts. Observable JavaScript semantics must not change at `-O3`.

| CLI level | Default safety mode | Contract |
|---|---|---|
| `-O0` | `safe` | Prefer debug visibility and direct lowering. |
| `-O1` | `safe` | Apply only local, obviously semantics-preserving rewrites. |
| `-O2` | `typed` | Use type/control-flow facts while preserving runtime checks. |
| `-O3` | `typed` or proven `strict-wasm` | Specialize only where guards prove semantic equivalence. |
| explicit `unsafe-fast` | `unsafe-fast` | Experimental mode; never the default and never implied by `-O*`. |

## Feature Priority

| Priority | Meaning |
|---|---|
| `P0` | Semantic bug, security/capability issue, or basic execution blocker. |
| `P1` | Widely used syntax, builtin, module, or WASI behavior. |
| `P2` | Advanced JS/TS feature, Stage 4+ wasm feature, or broader runtime parity. |
| `P3` | Legacy feature, future target, or optimization-only work. |
| `future` | Accepted as a possible direction but not in the current compatibility goal. |

## Workstreams And Gates

The project success metric is broad semantic compatibility, measured primarily through reference coverage and differential evidence. Workstreams are independent lanes; gates are evidence checkpoints, not task lists.

| ID | Lane | Evidence focus |
|---|---|---|
| W0 | Runtime substrate | Valid wasm, ABI invariants, runtime helper linkage. |
| W1 | Standalone WASI execution | Console/stdin/stdout without Node host power. |
| W2 | Syntax coverage | Parser/frontend ownership and unsupported diagnostics. |
| W3 | Name resolution | Scope, builtin identity, hoisting, and unresolved-name burn-down. |
| W4 | Builtin runtime | String, Array, Object, Date, RegExp, JSON, Promise, collections. |
| W5 | Runtime semantics | Completion records, classes, closures, modules, exceptions, async. |
| W6 | Reference coverage ramp | test262/tsc/tsgo selection, triage, and matrix freshness. |
| W7 | Host capability boundary | Manifest/link-plan equality and `--host-deny` behavior. |
| W8 | Optimization | Performance work that preserves the semantic gates above. |

Minimum local gate names are stable even when command wrappers change:

| Gate | Meaning | Typical command |
|---|---|---|
| A | formatter + Rust test suite | `python scripts/manager.py nextest` plus formatting wrapper |
| B | compiler diagnostics and phase contracts | `python scripts/manager.py check` |
| C | runtime/catalog/link-plan invariants | `python scripts/manager.py gate-fast` |
| D | coverage artifact freshness | `python scripts/manager.py coverage-report --format markdown` or matrix check wrapper |
| E-H | broader reference and semantic gates | documented by the active issue or release slice |
