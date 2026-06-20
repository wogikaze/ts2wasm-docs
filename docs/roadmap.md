# ts2wasm Roadmap

This roadmap describes work lanes. Concrete tasks live in `issues/`.

## Snapshot

- Issues under `issues/`: 466 files.
- Status: done=402, open=62, doing=2.
- Priorities: P2=212, P0=43, P3=22, P1=189.
- Fixture directories: 36 (pass=29, unknown=7).

## Lanes

| Lane | Focus | Success evidence |
|---|---|---|
| W0 Architecture | reduce coupling, file size, hidden runtime deps | architecture gates, focused refactor tests |
| W1 Frontend syntax | parse/erase/reject TS/JS syntax precisely | parser snapshots, type-erasure fixtures |
| W2 Resolution | lexical/module/builtin names | resolver snapshots, unresolved-name reduction |
| W3 IR | HIR/MIR validation and parity | dumps, MIR tests, compat fallback reduction |
| W4 Runtime builtins | Array/Object/String/Math/JSON/Date/RegExp/etc. | semantic/differential fixtures |
| W5 JS semantics | classes, async, modules, completion records | semantic canaries and targeted nextest |
| W6 Coverage/tooling | reference runner, dashboard, JSONL schema | coverage matrix, dashboard data, schema checks |
| W7 Targets/ABI | future wasm-gc/component paths | target gates and ABI snapshots |

## Rules

- Do not mark a lane complete from build coverage alone.
- Do not create broad issues without acceptance commands.
- Keep roadmap high-level; put implementation details in issues and feature docs.
