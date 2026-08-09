# Branded and Opaque Types

Use a branded or opaque type only for a new invariant that remains unsafe with the
project's current types. Reuse canonical IDs, units, validated values, parsers, and
smart constructors whenever they exist.

## Inspect before introducing a brand

Search the owning package and its dependencies for:

- the domain value or identifier;
- an installed library's brand, opaque, schema, or refinement support;
- canonical parsing and validation functions;
- serialization and database mappings;
- fixtures, generators, and test builders.

Do not create a second `UserId`, `Email`, money unit, timestamp unit, or parser to
solve a local typing inconvenience. Import the canonical type or improve it at its
owner when the current task requires that change.

## When a new brand is justified

A new brand is useful when all of these conditions hold:

1. The value has a distinct invariant or unit.
2. Mixing it with the underlying primitive can cause a real defect.
3. No existing project or library type represents it.
4. The owning boundary can validate or construct it consistently.

Prefer a discriminated union, enum, schema-derived type, or small domain object
when those better express the behavior. A brand should not stand in for missing
runtime validation.

## Construction and parsing

Use the project's installed branding or schema facility. Keep construction behind
the canonical parser or validator, and return the project's existing error type.
Avoid standalone generic `Brand` aliases, casts scattered through application
code, and duplicate smart-constructor libraries.

At trusted internal boundaries, follow the existing project convention for
constructing already validated values. At untrusted boundaries, validate before
the value enters the domain.

## Integration

- Preserve the public wire representation during internal migrations.
- Reuse database column types and mapping helpers already owned by the data layer.
- Reuse canonical fixtures and generators in tests.
- Keep serialization explicit when the library type is not JSON-native.

## Review checklist

- The brand protects a new, concrete invariant.
- No canonical type or parser already exists.
- Runtime validation and error behavior match the owning package.
- APIs, persistence, and fixtures continue to use one canonical representation.
