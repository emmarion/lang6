# lang6 Language Design

We are designing **lang6**, a statically typed, eagerly evaluated language with restricted dependent types, targeting low-level correctness-critical domains (crypto, compression, parsing).

## Key Documents

- `docs/00-spec.md` — design goals, silos, type system, termination, enums/unions, transmute, call modes, variable declarations
- `docs/01-open-questions.md` — unresolved design questions (arithmetic, memory model, FFI, error handling, modules, syntax, comptime)

## Core Design Principles

- No `unsafe` escape hatch — the type system must be strong enough to eliminate runtime checks
- Four silos: LR (linear runtime), LE (linear erased), UR (unrestricted runtime), UE (unrestricted erased — sink, all type expressions live here)
- No function application in types — only constructors. Proof types like `Add` bridge the gap
- Guaranteed termination via levels + lexicographic subterm ordering
- Explicit over magical — proofs, level ordering, and impossibility are programmer-visible
- `Type : Type` (believed consistent due to termination guarantee, needs verification)

## Current State

The goals doc captures the core type system, silos, termination mechanism, enums/unions, match branches, transmute rules, four call modes (lower/recur/tail/delegate), and variable declaration rules. Open questions remain for arithmetic semantics, memory model, FFI, error handling, modules, syntax, and comptime.

## Conventions

- When updating design docs, be succinct but complete
- All design decisions are the user's — offer feedback, suggestions, and critiques, but the user has final say
- We start broad, narrow down, and eventually plan to write a compiler (likely targeting C)
