# Migrating to Functional Patterns

Migrate incrementally from the owning package's current model. Functional
patterns are tools for a specific behavior change, not a reason to replace
working types or create a parallel utility layer.

## Establish the baseline

1. Inspect package dependencies, compiler settings, shared domain modules, and
   active boundaries.
2. Identify canonical Option-, Result-, error-, validation-, ID-, and brand-like
   types.
3. Record the contract that callers observe before changing internals.
4. Locate the concrete defect, unsafe invariant, or state-modeling problem that
   motivates the migration.

Do not begin by creating generic `option.ts`, `result.ts`, `brand.ts`, or
`errors.ts` files. Use installed libraries and existing project helpers.

## Choose the smallest change

- Replace related booleans with a discriminated union when invalid combinations
  are possible in the changed code.
- Use the existing absence type when a changed operation models expected absence.
- Use the existing failure type when a changed operation exposes recoverable
  errors.
- Introduce a brand only for a new invariant that cannot be protected safely with
  canonical project types.

Keep conversions at active boundaries. Preserve public APIs, persistence shapes,
and error behavior unless changing one is part of the task.

## Migration order

1. Add or update focused tests for the changed behavior and its active boundary.
2. Reuse the canonical type and helpers in the smallest affected unit.
3. Adapt immediate callers without spreading a second representation.
4. Remove obsolete local adapters once no current consumer needs them.
5. Type-check and run the owning package's focused tests.

Avoid repository-wide adoption schedules, new lint rules, CI gates, or training
work unless the task explicitly includes them.

## Review checklist

- The migration fixes a current modeling or safety problem.
- Existing library and project types were reused.
- No duplicate Result, Option, brand, parser, or error helper remains.
- Public and operational contracts remain stable unless explicitly changed.
- Tests cover changed behavior without repeating the same scenario at every layer.
