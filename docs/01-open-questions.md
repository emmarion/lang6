# lang6 — 01 — Open Design Questions

- **Arithmetic semantics**: lang6 rejects runtime overflow checks. How are fixed-width integers defined? How do you prove "this addition doesn't overflow" without type-level computation? Is there a proof type for it?
- **Memory model & allocation**: LR enables mutation, but concretely how? Stack vs heap? Manual free? How do pointers work and how does the "not freed" LE proof relate?
- **Foreign function interface**: targeting C requires calling C and being callable from C.
- **Error handling**: total functions can't fail, but external operations (IO, parsing) can. What replaces exceptions/panics?
- **Module system**: namespaces, visibility, separate compilation.
- **Syntax sketch**: enough notation to write examples — how are silos, levels, subterm annotations, and match kinds written?
- **Recursive types & Type : Type**: `Type : Type` with guaranteed termination via levels should be consistent, but this needs formal verification.
- **Comptime & levels interaction**: resolved — comptime doesn't run at a level; it operates above the level system via the `comptime` call mode, which requires no proof obligation. The callee's termination is already established by its own level and body.