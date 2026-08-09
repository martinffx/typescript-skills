---
name: typescript-functional-patterns
description: Functional programming patterns for reliable TypeScript. Use when modeling state machines, discriminated unions, Result/Option types, branded types, or building type-safe domain models.
user-invocable: false
---

# Functional Patterns for Reliable TypeScript

Inspect the owning package and existing implementation first. Reuse established
project types, helpers, errors, lifecycle behavior, and test utilities. The
patterns below are options, not an implementation checklist. Introduce one only
when the current task requires it.

## Project-specific rules

- Remove paste-ready implementations of Result, Option, brands, and error helpers.
- Reuse types supplied by installed libraries and the existing codebase.
- Introduce a branded type only for a new invariant that is otherwise unsafe.
- Do not create another branded ID or parser when a canonical one exists.

Build reliable systems using Algebraic Data Types (ADTs), discriminated unions, Result/Option types, and branded types. These patterns enable the compiler to prove correctness, prevent runtime errors, and make illegal states unrepresentable.

## Why Functional Patterns?

**Reliability through types**: Use the type system to encode business rules, making invalid states impossible to construct. The compiler becomes your safety net, catching errors at build time rather than runtime.

**Key benefits:**
- Exhaustiveness checking prevents missing cases
- Impossible states become unrepresentable
- Business logic encoded in types, not runtime checks
- Refactoring becomes safe and mechanical
- Self-documenting code through types

## Quick Reference

For detailed patterns and examples, see:
- [ADTs (Algebraic Data Types)](./references/adts.md) - Sum types, product types, discriminated unions
- [Option & Result](./references/option-result.md) - Type-safe error handling and nullable values
- [Branded Types](./references/branded-types.md) - Smart constructors and nominal typing
- [Migration Guide](./references/migration-guide.md) - Step-by-step adoption playbook

## Core Patterns Overview

### 1. Discriminated Unions (Sum Types)

Model "one of several variants" with exhaustive pattern matching:

```typescript
type PaymentMethod =
  | { kind: "card"; last4: string; brand: string }
  | { kind: "ach"; accountNumber: string; routingNumber: string }
  | { kind: "wallet"; provider: "apple" | "google" }

function processPayment(method: PaymentMethod): void {
  switch (method.kind) {
    case "card":
      // TypeScript knows: method.last4 and method.brand exist
      return processCard(method.last4, method.brand)
    case "ach":
      // TypeScript knows: method.accountNumber and method.routingNumber exist
      return processACH(method.accountNumber, method.routingNumber)
    case "wallet":
      // TypeScript knows: method.provider exists
      return processWallet(method.provider)
    default:
      assertNever(method) // Compiler error if cases missing
  }
}
```

### 2. Option Type (Nullable Values)

Use the project's existing nullable-value representation. If an installed library
already supplies `Option`, use its constructors, combinators, and matching APIs
instead of defining another type.

### 3. Result Type (Error Handling)

Use the project's established failure type for recoverable errors. Preserve its
error values and propagation conventions rather than adding a parallel `Result`
implementation or error hierarchy.

### 4. Branded Types (Type-Safe Units)

Use an existing project brand or opaque type when one already represents the
invariant. Add a brand only when a new invariant cannot otherwise be enforced
safely at the relevant boundary.

## When to Use

### Use Discriminated Unions When:
- Modeling state machines (pending → settled → reconciled)
- Representing mutually exclusive variants (payment methods, user roles)
- Building domain models with distinct states
- Replacing boolean flags with explicit states

### Use Option When:
- Value may be absent (but absence is expected/valid)
- Replacing `null` or `undefined` checks
- Chaining operations that may fail to find values
- Making nullability explicit in APIs

### Use Result When:
- Operation may fail with recoverable errors
- You need to propagate error context
- Replacing try/catch for expected failures
- Building error handling into function signatures

### Use Branded Types When:
- Preventing unit confusion (cents vs dollars, ms vs seconds)
- Enforcing validation invariants (email format, positive numbers)
- Introducing a new identifier invariant with no canonical project ID
- Domain-driven design with value objects

## Guidelines

### Pattern Matching Best Practices

1. Use the project's existing exhaustive matching convention, whether that is a
   native `never` check, a library matcher, or a canonical helper.

2. **Use discriminant field consistently** (`kind`, `type`, `_tag`):
   ```typescript
   type Status = { kind: "ready" } | { kind: "blocked"; reason: string }
   ```

3. **Narrow types early** to unlock type safety:
   ```typescript
   if (result._tag === "Ok") {
     // TypeScript knows: result.value exists
     return result.value.data
   }
   ```

### Error Handling Strategy

1. Use the existing Option-like type for expected absence.
2. Use the existing Result-like type for recoverable errors.
3. Preserve the project's exception and defect conventions for programmer errors.

### Branded Types Guidelines

1. Reuse canonical brands and their parsers or smart constructors.
2. Add a brand only for a new invariant that primitive typing cannot protect.
3. Validate new brands at the boundary where untrusted values enter.

### Migration Strategy

Keep the migration inside the current task. Reuse canonical types in the changed
code, update its immediate consumers, and preserve existing compiler settings and
public contracts unless the task explicitly changes them.

## Examples by Domain

### State Machine (Transaction Lifecycle)
```typescript
type TxnState =
  | { kind: "pending"; createdAt: Date }
  | { kind: "settled"; ledgerId: LedgerId; settledAt: Date }
  | { kind: "failed"; reason: FailureReason; failedAt: Date }
  | { kind: "reversed"; originalLedgerId: LedgerId; reversedAt: Date }

function canReverse(state: TxnState): boolean {
  switch (state.kind) {
    case "pending": return false
    case "settled": return true
    case "failed": return false
    case "reversed": return false
    default: assertNever(state)
  }
}
```

### Configuration Parsing

Parse configuration with the validation library and error type already used by
the owning package. Keep its existing return shape and error reporting contract.

### Financial Calculations

Reuse the project's canonical money type and arithmetic helpers. Add a unit type
only when the codebase lacks one and mixing the underlying primitives remains an
active safety risk.

## Further Reading

- [ADT Reference](./references/adts.md) - Deep dive on sum types, product types, and pattern matching
- [Option & Result Reference](./references/option-result.md) - Comprehensive error handling patterns
- [Branded Types Reference](./references/branded-types.md) - Advanced nominal typing techniques
- [Migration Guide](./references/migration-guide.md) - Step-by-step adoption playbook

## Credits

These patterns are inspired by **[Why Reliability Demands Functional Programming, ADTs, Safety and Critical Infrastructure](https://rastrian.com/why-reliability-demands-functional-programming-adts-safety-and-critical-infrastructure/)** by Rastrian. The blog post explores how functional programming techniques and Algebraic Data Types enable building reliable systems in critical infrastructure contexts.

## When This Skill Loads

This skill automatically loads when discussing:
- Discriminated unions and sum types
- State machine modeling
- Result/Option types and error handling
- Branded types and smart constructors
- Type-safe domain models
- Making illegal states unrepresentable
- Functional programming in TypeScript
