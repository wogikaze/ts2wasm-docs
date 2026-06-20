# Object Semantics Kernel

## Scope

This document covers object/property/prototype semantics implemented through runtime helpers and heap object layout.

## Runtime families

Object helpers include property get/set/delete/has, own keys/names/symbols, descriptors, freeze/seal/preventExtensions, prototype operations, assign/create/fromEntries, Object.is, instanceof, valueOf/toString/toLocaleString, and descriptor attribute tracking.

## Heap object layout

Objects use a header with property count, flags, prototype pointer, and property entries. Flags track frozen/sealed/non-extensible and limited per-property attributes. Constants live in `Layout`.

## Rules

- Property semantics changes must update runtime helpers, fixtures, and object-specific tests.
- Attribute/prototype behavior must not be implemented as ad-hoc string checks in backend emission.
- If a runtime helper needs new strings/imports/capabilities, declare them in `RuntimeFn` spec.

## Tests

Relevant tests include `crates/cli/tests/object_kernel.rs`, `object_methods.rs`, `object_descriptors.rs`, and object fixtures in `fixtures/arrays-objects` and `fixtures/builtins-and-io`.
