# lang6 — 00 — Goals

## Motivation

Rust forces `unsafe` in the **implementation** of fundamental data structures (Vec, HashMap) because its safe subset isn't expressive enough. It also papers over gaps with runtime checks (integer overflow, bounds checks) that differ between debug and release. This language aims to eliminate both: the type system should be strong enough that `unsafe` isn't needed, and invariants should be proven statically so no runtime checks are necessary.

The target domain is low-level, correctness-critical code: encryption, compression, file format parsing.

## Hard Goals

- Statically typed, eagerly evaluated
- Restricted dependent types: ADTs/GADTs can appear in types (see Type System)
- Dependent pattern matching with Axiom K and unification (Idris 2 style)
- Four silos: LR (linear runtime), LE (linear erased), UR (unrestricted runtime), UE (unrestricted erased)
- "Linear" = linear + unique — exactly one use AND no aliasing, enabling safe in-place mutation
- No runtime checks — invariants proven statically
- Guaranteed termination via levels + lexicographic subterm ordering
- No currying — all arguments explicit
- Enums (runtime discriminant) and unions (erased discriminant)
- Functions and match expressions can return zero or more values (not just one)
- Every variable declaration must explicitly provide its silo
- Every value constructor invocation must explicitly specify the silo of the constructed value
- Every function invocation must explicitly specify the silo of the call
- Every match expression must explicitly specify the mode of the match
- Every function definition and match expression must have an explicitly annotated return type
- **"is" declarations** can appear wherever a type annotation is expected (variable declarations, function parameters, constructor fields, etc.): the binding is declared equal to a provided expression (which follows the no-function-application rule from the Type System section). The actual type is derivable from the expression. The silo of the binding can differ from the silo of the expression it equals. "is" replaces only the type — the initializing expression is still required (when relevant).
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
- **Explicit over magical**: termination proofs, level ordering, and absurdity proofs are programmer-visible, not inferred behind the scenes.
- **No hidden cost**: if something is erased, it has zero runtime representation.
- **UR-projection soundness**: Any well-typed lang6 program, with all silo annotations replaced by UR and phantom designations ignored, is denotationally equivalent — it produces the same logical results — except at FFI boundaries. Operationally, unions collapse to enums, transmutes become copies, tail calls may lose tail optimization, and delegate wrappers execute normally; these preserve results but not resource guarantees. This property constrains the design: features must not introduce denotationally observable behavior that depends on whether computation happens at runtime or is erased, unless that observation crosses an FFI boundary.

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



Applied at four sites:

- **Construction**: argument provided at `meet(constructor_silo, field_declared_silo)`, where constructor_silo is the silo specified at the invocation site.
- **Matching (pattern bindings)**: depends on match mode — erased: UE; peek: UE; normal: `meet(scrutinee_silo, field_declared_silo)`, where scrutinee_silo is the silo of the value being matched.
- **Function invocation (arguments)**: argument provided at `meet(call_silo, param_declared_silo)`, where call_silo is the silo specified at the call site.
- **Function invocation (return values)**: return value assigned `meet(call_silo, return_declared_silo)`, where call_silo is the silo specified at the call site (when there are return values).

Match return values use their declared silos directly (subject to phantom-constructor erasure rules).

The silo of a call determines whether the body executes at runtime (LR/UR: yes; LE/UE: no). This matters even for zero-return-value invocations (e.g., linear destructors).

A value's silo is fixed at construction and does not change for the lifetime of the value. The only silo transition is `erase`, which produces a UE expression definitionally equal to the original — the original retains its silo.

ADT types are not defined at a particular silo — the same type can be constructed at any silo. The silo is specified at each constructor invocation site, not in the type definition. Declared field silos are ceilings: the effective silo of a field at a given site is `meet(invocation_silo, declared_silo)`, which may be weaker than the declared silo.

Certain base types (Level, LevelGT, etc.) are always UE.

## Enums, Unions, and Matching

**Enums** have a runtime discriminant. **Unions** have an erased (UE) discriminant.

**Match modes** — a match expression specifies one of three modes:

| Mode | Scrutinee | Pattern bindings | Discriminant availability |
|------|-----------|-------------------|---------------------------|
| **Erased** | Treated as UE copy; not consumed | UE | Unavailable |
| **Peek** | Not consumed | UE | Per scrutinee: enum at LR/UR → available; else → unavailable |
| **Normal** | Consumed (if linear) | `meet(scrutinee_silo, field_declared_silo)` | Per scrutinee: enum at LR/UR → available; else → unavailable |

**Live branch count** — how many Live branches (non-phantom, non-absurd) are permitted:

- Discriminant available → any number of Live branches.
- Discriminant not available + any non-phantom branch has code annotated with non-erased silos → exactly one Live branch.
- Discriminant not available + all non-phantom branches contain only code annotated with erased silos → any number of Live branches.

Note: phantom constructor branches currently require all code to be annotated with erased silos, so "non-phantom" is technically redundant as the spec stands. The qualifier is included in case the spec changes to permit code annotated with non-erased silos in phantom branches.

**Linearity**: per-branch discipline. Each branch must independently consume its linear bindings exactly once. The match mode determines whether and how the scrutinee is consumed; after that, normal linearity rules apply.

A match expression requires an explicit mode keyword. Note: for type-correctness across all invocation silos, phantom constructors must still be accounted for in pattern matches (see Phantom constructors).

**Match branches** come in four kinds:

| Kind | Body | Absurdity via |
|------|------|---------------|
| Live | Produces values of the match's return type | — (branch may run) |
| Phantom constructor | All variables in scope treated as erased: UR→UE, LR→LE, UE and LE unchanged; return type also treated as erased | — |
| Absurd | No body | Compiler unification alone |
| Proved absurd | Produces a value of an empty type (e.g., `False`) using pattern variables | Programmer constructs proof |

TODO: May change — we might just ignore erasureness here in the future.

## Type System

- **ADTs and GADTs**: types can be indexed by constructor terms.
- **No function application in type expressions**: type expressions and `is` expressions may contain type constructors, value constructors, and variable references — but no function applications. `Vec (S n)` is legal; `Vec (add n 1)` is not. Proof types like `Add : Nat → Nat → Nat → Type` bridge this gap.
- **Uninhabited types are useful**: `False`, `Infinite`, etc. serve as absurdity evidence and type-level constraints even though they have no inhabitants.
- **Pattern matching unifies**: matching a GADT refines types in scope via unification (Axiom K).
- **Constructors have no levels**: value and type constructors are always available regardless of level. Only function definitions carry level annotations.
- **Phantom constructors**: a constructor may be declared phantom. Phantom constructors can only be constructed at erased silos (LE/UE). The true silo of a function argument is the meet of the entire invocation chain and its declared silo, so even a declared UR/LR ADT value may have a phantom constructor as its discriminant when the invocation chain is erased. Since erased code must be type-correct — other code may depend on proofs constructed in erased contexts — pattern matches must account for phantom constructors as possible discriminants. Code cannot assume a declared UR/LR ADT value was not constructed via a phantom constructor.
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
| Newtype wrapping | Single constructor with exactly one LR/UR argument | Transmute subproofs for all LE arguments; UE arguments need no proof. For enums: exactly one constructor total. For unions: every other constructor stores no runtime data (is phantom, or has only LE/UE declared arguments, or both), and the programmer provides a constructor expression that the compiler checks for definitional equality with the scrutinee, establishing which constructor form the value takes |
| Same constructor | Two values share a constructor | UE arguments freely differ; LR, LE, and UR arguments require transmute proofs |
| Level lifting | `LevelGT a b` (both UE) + function pointer at level `b` | Produces function pointer at level `a` + transmute proof. Safe because a lower-level function can always be called from a higher level |
| Delegate unwrapping | Function defined via `delegate` to another | Proof from delegator to delegatee. `delegate` guarantees the body is exactly a call to the target after removing LE/UE code and transmute wrappers, so the two are representationally identical |

## Termination

Every function carries **level** and **subterm** annotations in its type.

**Four call modes:**

In all four call modes, the invoked function may be either a named function or a function pointer. Function pointers are always unrestricted (UR or UE — never linear). Function pointers are likely "fat": one entry point for UR invocation silo and one for LR invocation silo, since the generated code differs by invocation silo.

- **`lower`**: calls a function at a strictly lower level. Requires a `LevelGT : Level → Level → Type` proof, provided via `lvl-gt` special form (compiler verifies, errors if it can't prove the ordering).
- **`recur`**: calls a function at the same level. Requires lexicographic descent:
  - Subterm-annotated arguments are ordered.
  - Arguments before the first difference must be equal (proven by definitional equality).
  - The first differing argument must be a strict subterm (proven via `Subterm`).
  - Arguments after that are unconstrained.
- **`tail`**: like `recur` (same level, lexicographic descent), but the compiler verifies the call is in tail position. After removing any erased code and transmutes, the return result must match what is eventually returned. This enables tail-call optimization with a termination guarantee.
- **`delegate`**: the entire function body, after transmutation removal and erased-code removal, must be exactly a call to the invoked function. Like `lower`, the called function can be at a lower level. The function is a pure wrapper — no computation beyond delegation.

Each call mode requires exactly one proof: `lower` and `delegate` require a `LevelGT` proof, while `recur` and `tail` require a `Subterm` proof (for the first differing argument). A single call never needs both kinds.

**Subterm proofs:**

```
Subterm : (A: Type) → (a: A) → (B: Type) → (b: B) → Type
```

- Subterm arguments can be in any silo (LR, LE, UR, or UE).
- `prove-subterm a b` — special form, compiler checks via definitional equality, errors if unprovable. Cross-type: `A` and `B` may differ.
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
- `def-lvl` may appear inside functions, dynamically creating new levels. These levels are UE and can be captured by closures.

**Closures (v1 restriction):**

Closures may only close over UR and UE values. Closing over LR and LE values is deferred to v2.

## Future (v2)

- "Everything linear or erased" — eliminate UR, making the language purely linear with phantom annotations. Acknowledged but deferred.

## Comptime and Macros

Placeholder. The language will support compile-time computation and macros (à la Zig/Racket). Since the language guarantees termination, comptime always terminates. Details TBD.

