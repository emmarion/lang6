# lang6 — 00 — Goals

## Motivation

Rust forces `unsafe` in the **implementation** of fundamental data structures (Vec, HashMap) because its safe subset isn't expressive enough. It also papers over gaps with runtime checks (integer overflow, bounds checks) that differ between debug and release. This language aims to eliminate both: the type system should be strong enough that `unsafe` isn't needed, and invariants should be proven statically so no runtime checks are necessary.

The target domain is low-level, correctness-critical code: encryption, compression, file format parsing.

## Hard Goals

- Statically typed, eagerly evaluated
- Restricted dependent types: ADTs/GADTs can appear in types (see Type System)
- Dependent pattern matching with Axiom K and unification (Idris 2 style)
- Four silos: LR (linear runtime), LE (linear erased), UR (unrestricted runtime), UE (unrestricted erased)
- Linear = exactly one use + unique reference (enabling safe in-place mutation)
- No runtime checks — invariants proven statically
- Guaranteed termination via levels + lexicographic subterm ordering
- No currying — all arguments explicit
- Enums (runtime discriminant) and unions (erased discriminant)
- Functions and match expressions can return multiple values (product/sigma types)
- Every variable declaration must explicitly provide its silo
- Every value constructor invocation must explicitly specify the silo of the constructed value
- Every function definition and match expression must have an explicitly annotated return type
- Variables can use **"is" declarations** instead of type declarations: the variable is declared equal to a provided expression (which follows the constructor-only rule from the Type System section). The actual type is derivable from the expression. The silo of the variable can differ from the silo of the expression it equals.
- Compile-time computation and macros (details TBD)

## Non-Goals

- HoTT or propositional equality beyond Axiom K
- Full dependent types with type-level computation
- General-purpose ergonomics (syntax is a low priority)
- Currying or partial application
- Runtime checks in any mode

## Design Principles

- **Proofs as resources**: LE values track protocol obligations; UE values express closed facts. Erased references don't consume linear bindings.
- **Erasure discipline**: UE is free to reference anything — references in type expressions never count as consumption.
- **Explicit over magical**: termination proofs, level ordering, and impossibility proofs are programmer-visible, not inferred behind the scenes.
- **No hidden cost**: if something is erased, it has zero runtime representation.

## The Four Silos

| Silo | Must consume | Runtime rep | Use |
|------|-------------|-------------|-----|
| **LR** | Exactly once | Yes | Buffered data, file handles |
| **LE** | Exactly once | No | Capability tokens, "not freed" proofs |
| **UR** | Freely | Yes | Ordinary values (Nat, String) |
| **UE** | Freely | No | Type-level indices, closed proofs |

Transition rules:
- **Erase** (special form): `erase x` produces a UE expression that is definitionally equal to `x` (recognized by the canonicalization rewrite system). The original value is not consumed and retains its original silo. `erase` is the only built-in silo transition.
- UE is the only free projection target. LR, LE, and UR are otherwise **independent** — no built-in way to move between them.
- All type expressions live in UE.

**Silo meet** — when constructing or matching a value of invocation silo X with a field of declared silo Y, the effective silo is `meet(X, Y)`:

| meet(invocation, declared) | **LR** | **LE** | **UR** | **UE** |
|---|---|---|---|---|
| **LR** | LR | LE | UR | UE |
| **LE** | LE | LE | UE | UE |
| **UR** | UR | UE | UR | UE |
| **UE** | UE | UE | UE | UE |

- **Runtime**: result has runtime rep only if both are runtime.
- **Linearity**: result is linear only if both are linear.

Applied at two sites:

- **Construction**: argument provided at `meet(invocation_silo, declared_silo)`, where invocation silo is the silo of the constructed value.
- **Matching**: binding assigned `meet(invocation_silo, declared_silo)`, where invocation silo is the silo of the matched value.

A value's silo is fixed at construction and does not change for the lifetime of the value. The only silo transition is `erase`, which produces a UE expression definitionally equal to the original — the original retains its silo.

ADT types are not defined at a particular silo — the same type can be constructed at any silo. The silo is specified at each constructor invocation site, not in the type definition. Declared field silos are ceilings: the effective silo of a field at a given site is `meet(invocation_silo, declared_silo)`, which may be weaker than the declared silo.

Function pointers are always unrestricted (UR or UE — never linear). Certain base types (Level, LevelGT, etc.) are always UE.

## Enums and Unions

**Enums** have a runtime discriminant. Branching is a runtime operation.

**Unions** have an erased discriminant. Matching on a union requires either:
1. The matched value is at an erased silo (LE or UE — use `erase` to obtain a UE expression definitionally equal to the original), or
2. All branches except one are proven impossible.

**Match branches** come in three kinds:

| Kind | Body | Impossibility via |
|------|------|-------------------|
| Normal | Produces match return type | — (branch may run) |
| Absurd | No body | Compiler unification alone |
| Proved impossible | Produces a value of an empty type (e.g., `False`) using pattern variables | Programmer constructs proof |

The silo of a match expression is determined by the silo of the value being matched — no separate silo annotation is needed. Matching a UE value is phantom; matching an LR/UR value is runtime.

## Type System

- **ADTs and GADTs**: types can be indexed by constructor terms.
- **Types contain constructors only**: `Vec (S n)` is legal; `Vec (add n 1)` is not. Proof types like `Add : Nat → Nat → Nat → Type` bridge this gap.
- **Uninhabited types are useful**: `False`, `Infinite`, etc. serve as impossibility evidence and type-level constraints even though they have no inhabitants.
- **Pattern matching unifies**: matching a GADT refines types in scope via unification (Axiom K).
- **Constructors have no levels**: value and type constructors are always available regardless of level. Only function definitions carry level annotations.
- **Phantom constructors**: a constructor may be declared phantom. Phantom constructors can only be constructed in an erased context (LE or UE silo). However, since any ordinary function can be invoked in a phantom context, code cannot assume that a value was not constructed via a phantom constructor.
- **Recursive types are allowed** with no strict positivity requirement.
- **`Type : Type`**: the language allows `Type : Type`. This would normally be inconsistent (Girard's paradox), but the termination guarantee via levels should prevent the circular constructions needed for the paradox. This requires careful metatheoretic verification.
- **Transmute**: safe type-punning via proofs.

```
Transmute : Linearity → (A: Type) → (a: A) → Linearity → (B: Type) → (b: B) → Type
```

A `Transmute` proof (usually phantom nonlinear) demonstrates that `a` can be transmuted to `b` and vice versa. A special form performs the actual runtime transmutation given a proof.

**Built-in functions:**

- **`transmute-refl`**: produces `Transmute L A a L A a`
- **`transmute-sym`**: `Transmute L A a L' B b → Transmute L' B b L A a`
- **`transmute-trans`**: `Transmute L A a L' B b → Transmute L' B b L'' C c → Transmute L A a L'' C c`

**Construction rules:**

| Rule | When | Requirement |
|------|------|-------------|
| Newtype wrapping | Single constructor with exactly one LR/UR argument | Transmute subproofs for all LE arguments; UE arguments need no proof. For enums: exactly one constructor total. For unions: every other constructor stores no runtime data (is phantom, or has only LE/UE declared arguments, or both), and the value must obviously come from that constructor |
| Same constructor | Two values share a constructor | UE arguments freely differ; LR, LE, and UR arguments require transmute proofs |
| Level lifting | `LevelGT a b` (both UE) + function pointer at level `b` | Produces function pointer at level `a` + transmute proof. Safe because a lower-level function can always be called from a higher level |
| Delegate unwrapping | Function defined via `delegate` to another | Proof from delegator to delegatee. `delegate` guarantees the body is exactly a call to the target after removing LE/UE code and transmute wrappers, so the two are representationally identical |

## Termination

Every function carries **level** and **subterm** annotations in its type.

**Four call modes:**

- **`lower`**: calls a function at a strictly lower level. Requires a `LevelGT : Level → Level → Type` proof, provided via `lvl-gt` special form (compiler verifies, errors if it can't prove the ordering).
- **`recur`**: calls a function at the same level. Requires lexicographic descent:
  - Subterm-annotated arguments are ordered.
  - Arguments before the first difference must be equal (proven by unification).
  - The first differing argument must be a strict subterm (proven via `Subterm`).
  - Arguments after that are unconstrained.
- **`tail`**: like `recur` (same level, lexicographic descent), but the compiler verifies the call is in tail position. After removing any phantom code and transmutes, the return result must match what is eventually returned. This enables tail-call optimization with a termination guarantee.
- **`delegate`**: the entire function body, after transmutation removal and phantom removal, must be exactly a call to the invoked function. Like `lower`, the called function can be at a lower level. The function is a pure wrapper — no computation beyond delegation.

Each call mode requires exactly one proof: `lower` and `delegate` require a `LevelGT` proof, while `recur` and `tail` require a `Subterm` proof (for the first differing argument). A single call never needs both kinds.

**Subterm proofs:**

```
Subterm : (A: Type) → (a: A) → (B: Type) → (b: B) → Type
```

- Subterm arguments can be in any silo (LR, LE, UR, or UE).
- `prove-subterm a b` — special form, compiler checks via unification, errors if unprovable. Cross-type: `A` and `B` may differ.
- `subterm-trans` — compiler-provided function, composes two Subterm proofs transitively.

**Level definitions and proofs:**

```
LevelGT : Level → Level → Type
```

- **`def-lvl n`** — special form that creates a new erased `n : Level`. Every variable in scope at the time of the `def-lvl` has a lower level than `n`. Semantically, this is a global counter increment; the compiler never computes the actual integer.
- **`lvl-gt l1 l2`** — special form that produces a `LevelGT l1 l2` proof. The compiler verifies that `l1` was introduced by a `def-lvl` and `l2` was in scope at that point. Fails with a type error if the ordering cannot be established from scoping.
- Level transitivity is available: if `LevelGT l1 l2` and `LevelGT l2 l3`, then `LevelGT l1 l3`. `Level` and `LevelGT` are first-class types and values.
- **Function definitions** explicitly specify their level, which must already exist (from a prior `def-lvl`).
- **Higher levels call lower levels**: `lower`-calls go strictly downward. Library code defines low levels; `main()` sits high enough to reach everything it needs.
- **Level unreachability**: any level higher than `main()`'s level is unreachable at runtime. This is a language-level guarantee, not just an optimization — functions and closures at those levels are replaced with no-ops (relevant to type sizing and size monomorphization).
- `def-lvl` may appear inside functions, dynamically creating new levels. This interacts with closures (see below).

**Closures (v1 restriction):**

Closures may only close over non-linear values (erased or not). Closing over linear values is deferred to v2.

## Future (v2)

- "Everything linear or erased" — eliminate UR, making the language purely linear with phantom annotations. Acknowledged but deferred.

## Comptime and Macros

Placeholder. The language will support compile-time computation and macros (à la Zig/Racket). Since the language guarantees termination, comptime always terminates. Details TBD.

