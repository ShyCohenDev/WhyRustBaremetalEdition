# Should We Switch?

This chapter weighs the reasons for and against moving our bare-metal code from C to Rust, based on the three technical chapters.

## Reasons for

1. **Whole classes of bugs become compile errors.** Hardware conflicts, unprotected data shared with interrupts, DMA buffer misuse, out-of-bounds access, ignored errors. In C these are found on the bench or in the field. ([Ch. 1](technical/1-hardware-access.md), [2](technical/2-interrupts-and-shared-state.md), [3](technical/3-memory-safety.md))
2. **One language for the whole company.** We already use Rust on the OS side. Developers would know one language, not two.
3. **No more code written twice.** Shared logic becomes one `no_std` library used by both bare-metal and OS code.
4. **Better daily tooling.** One command to build, flash and see logs; built-in unit tests; reproducible builds.
5. **Same runtime cost.** No runtime and no garbage collector; most safety checks happen at compile time.

## Reasons against

1. **Learning curve.** Ownership, borrowing and generics take C developers time to learn. Bare-metal Rust adds its own patterns (typestate, critical-section tokens).
2. **Chip support varies.** Excellent for some chip families, partial for others. Missing drivers must be written by us.
3. **Libraries are mostly community-maintained**, not vendor-maintained, and many are below version 1.0.
4. **Fewer certified components.** Qualified compilers exist (Ferrocene), but certified RTOSes and middleware are mostly C.
5. **Migration cost.** Existing C code is tested and working. Rewriting it costs time and carries the risk of new bugs.
6. **Some things are more verbose.** For example sharing data with interrupt handlers, and long type names in HAL code.

## Recommendation

Move to Rust for new bare-metal code: it turns whole classes of firmware bugs into compile errors at the same runtime cost, and lets the company work in one language.
