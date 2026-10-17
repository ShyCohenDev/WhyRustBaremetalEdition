# Technical Part: Topics to Cover

Planning resource for the technical part of the book. This is not book content.

Narrowed in Phase 3 to the 5 concepts where the C/Rust difference matters most. Everything else (boot, linker scripts, timers, buses, performance) appears only where one of these needs it.

- **Audience:** technical, but may only vaguely know Rust and bare-metal. Explain each concept where it is first needed. No general Rust tutorial: link to the official docs.
- **Structure per concept:** the concept, how it is done in C, how it is done in Rust, honest limits, and a "C vs Rust at a glance" table.
- **Reference hardware:** ST Nucleo-F401RE (STM32F401, Cortex-M4).

## 1. Hardware Access and Peripheral Ownership

- Bare-metal in one page: memory map, registers, memory-mapped I/O
- Registers and `volatile`: CMSIS headers vs PACs generated from SVD
- Peripherals as owned singletons (`take()`), moves, zero-sized types
- Typestate: pin modes in the type; C vendor HAL vs Rust HAL blinky
- Frozen clock configuration passed to drivers
- `embedded-hal` portable drivers
- Limits: HAL coverage, `unsafe` bypass, verbosity

## 2. Interrupts and Shared State

- What an interrupt is, the race condition on `counter++`
- C: weak-symbol handlers, `volatile`, manual critical sections
- Rust: checked handler names, `static mut` needs `unsafe`, atomics, `critical_section::Mutex` token
- DMA: buffer ownership during a transfer, `'static` buffers
- RTIC and Embassy, and running Rust on an RTOS

## 3. Memory Safety, Errors and `unsafe`

- RAM layout (globals, stack), why bare-metal bugs are silent
- Out of bounds, NULL at address 0, signed overflow, ignored errors, dangling pointers
- Bounds checks, `Option`, defined overflow, `Result`, panics and panic handlers
- What Rust does not catch: stack overflow (`flip-link`), logic bugs
- No heap unless requested; `heapless`
- `unsafe`: the five operations, safe wrappers, `SAFETY` comments, Miri

## 4. C Interop and Code Sharing

- The C ABI and FFI
- Rust calling C: `extern "C"`, safe wrappers, `bindgen`, `cc`
- C calling Rust: `#[no_mangle]`, `staticlib`, `cbindgen`, Corrosion in CMake
- Data across the boundary: `#[repr(C)]`, enums, ownership, panics, C++
- One `#![no_std]` library for bare-metal, OS and remaining C firmware

## 5. Tooling and Ecosystem

- Cross-compiling: one compiler, targets, pinning, supply chain
- Startup code and memory layout: `cortex-m-rt` + `memory.x`
- Flashing and debugging with `probe-rs`
- Logging with `defmt`
- Testing: `cargo test`, `embedded-hal-mock`, on-target tests
- Binary size and speed (measurements to be done)
- Chip and vendor support, safety certification (Ferrocene), long-term maintenance

## Open Questions

- Do we include RISC-V examples, or only mention where it differs?
- Which concurrency model (bare loop / RTIC / Embassy / RTOS) do we recommend, or do we only compare?
- Which real firmware module do we port and measure for the size/speed table?
