# Coverage Expansion Epics

This document groups coverage expansion work into lanes. Individual executable tasks remain in `issues/`.

## Active lanes

| Lane | Goal | Evidence |
|---|---|---|
| Builtin API expansion | Reduce unsupported builtin classifications across Array/Object/String/Math/JSON/Date/RegExp/etc. | fixture + reference coverage deltas |
| Name resolution | Reduce unresolved-name/function classifications without over-allowing globals. | resolver snapshots, reference unsupported table |
| Classes/prototypes | Improve class fields, private members, prototype/descriptor behavior. | class fixtures and object kernel tests |
| Async/Promise | Improve async-await and Promise helper semantics. | async fixtures and `async_await` |
| Modules | Improve static imports/exports, module graph, require/export runtime. | module fixtures and module graph tests |
| TypeScript erasure | Parse/erase supported TS syntax and reject unsupported forms precisely. | type-erasure fixtures, tsc/tsgo coverage |

## Execution model

1. Read `docs/current-state.md` and generated coverage matrix.
2. Pick a small `issues/` item or create one with acceptance commands.
3. Add focused fixtures/tests first when possible.
4. Run focused gate, then relevant coverage slice.
5. Update docs only for changed contracts.

## Do not do

- Do not auto-create dozens of issues from one coverage table without triage.
- Do not mark semantic support complete from build coverage only.
