# Option and Result

Use this guide only after inspecting the owning package for an existing Option-
or Result-like type. Installed libraries and canonical project types take
precedence over the patterns described here.

## Choose the existing representation

Search imports, shared domain modules, validation code, and dependency manifests.
Common representations include library `Option`, `Either`, `Result`, or `Effect`
types; discriminated unions owned by the project; and ordinary `undefined` or
exceptions where those are the established contract.

Do not add another generic Option or Result implementation merely to make an API
look functional. A second representation adds adapters, duplicate helpers, and
new error conventions.

## Option-like values

Use the project's existing absence representation when absence is expected and
does not need an error payload. Preserve its constructors, combinators, matching
style, and boundary conversions.

Prefer the existing public return type during migrations. Convert internally only
when the task requires it, and keep the conversion at a clear boundary.

## Result-like values

Use the project's existing failure representation for recoverable failures. Reuse
canonical error values and preserve the established generic parameter order,
matching APIs, and propagation style.

Before changing a throwing API to return a Result-like value, confirm that the
task changes that contract. An internal migration must not silently change how
callers observe failure.

## Boundaries

- Decode untrusted input with the package's existing validation tools.
- Convert nullable, throwing, or promise-based APIs once at the owning boundary.
- Keep HTTP, database, queue, and public-library contracts unchanged unless the
  task explicitly changes them.
- Avoid wrappers that only rename library constructors or combinators.

## Selection guide

- Expected absence without details: use the established Option-like or nullable
  representation.
- Recoverable failure with details: use the established Result-like or typed-error
  representation.
- Programmer error or broken invariant: follow the project's defect or exception
  convention.
- Multiple validation failures: use the validation library's existing issue type
  and accumulation behavior.

## Review checklist

- The owning package and its dependencies were inspected first.
- No duplicate Option, Result, constructor, parser, or error helper was added.
- Existing callers observe the same contract unless the task changes it.
- New conversions exist only at active boundaries.
