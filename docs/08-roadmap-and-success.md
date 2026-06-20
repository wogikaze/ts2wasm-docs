# Roadmap and Success Criteria

## Success model

The project succeeds when it can compile a broad practical JS/TS subset to valid, auditable wasm with predictable unsupported diagnostics for the rest.

Success requires:

- increasing semantic and differential pass rates,
- decreasing unsupported/fail/blocked classifications,
- preserving standalone vs host-shim capability boundaries,
- keeping compiler phase contracts inspectable and tested,
- retaining docs that explain Why, not only What.

## Roadmap lanes

| Lane | Theme | Current docs |
|---|---|---|
| W0 | Architecture hygiene and maintainability | `docs/24-architecture-decoupling-and-llm-friendly-sizing.md` |
| W1-W3 | Frontend, name resolution, IR foundations | `docs/28-frontend-syntax-ownership.md`, `docs/13-ir-contracts.md` |
| W4 | Builtins/runtime expansion | `docs/05-compatibility-and-semantics.md`, `docs/14-runtime-abi.md` |
| W5 | JS runtime semantics: classes, async, modules | `docs/20-async-await-design.md`, `docs/21-object-semantics-kernel.md` |
| W6 | Coverage infrastructure and reference suites | `docs/06-testing-and-coverage.md`, `docs/23-coverage-runner-completeness.md` |
| W7+ | Future targets and ABI evolution | `docs/02-execution-model-and-targets.md`, `docs/14-runtime-abi.md` |

## Do not use this doc as a task list

Concrete work items live in `issues/`. This roadmap describes lanes and success criteria. When a lane becomes actionable, create or update an issue with acceptance commands and evidence requirements.
