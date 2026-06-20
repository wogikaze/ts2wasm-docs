# Async/Await Design

## Current intent

Async/await support is implemented as a runtime-backed subset using Promise and task helpers. The goal is practical fixture support without pretending to implement the full ECMAScript job queue yet.

## Runtime pieces

Relevant RuntimeFn families include:

- `PromiseConstructor`, `PromiseResolve`, `PromiseReject`, `PromiseThen`, `PromiseCatch`, `PromiseFinally`, `PromiseAll`, `PromiseAllSettled`, `PromiseAny`, `PromiseRace`, `PromiseWithResolvers`
- `TaskPoll`, `TaskResult`, `TaskDrop`
- async fixture support in `fixtures/async-await`

## Semantic constraints

- Async behavior must preserve observable output for covered fixtures.
- Unsupported scheduling/microtask semantics should be explicit rather than silently approximated.
- Host-backed behavior must be reflected in capability/link-plan docs when introduced.

## Tests

Start with:

```bash
cargo nextest run -p ts2wasm-cli --test async_await
```

Then use fixture/differential gates as needed.
