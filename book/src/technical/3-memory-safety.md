# Memory Safety, Errors and `unsafe`

On a PC, the operating system stops a program that accesses bad memory ("segmentation fault"). On bare-metal nothing stops it: a memory bug silently corrupts another variable, and the device fails later, somewhere unrelated.

## RAM in short

Global variables sit at the start of RAM. The **stack** (local variables and function calls) starts at the end of RAM and grows down toward them. Most firmware has no **heap** (`malloc`), because allocation failures and fragmentation are hard to handle on a device nobody watches.

## Silent in C, caught in Rust

| Bug | C on a microcontroller | Rust |
|---|---|---|
| Index past the end of an array | Overwrites the next variable | Stops with a **panic** (file and line), or `buf.get(i)` returns "nothing" |
| NULL pointer read | Address 0 is flash, so the read **succeeds** and returns code as data | No NULL: a possibly missing value has type `Option` and must be checked |
| Signed integer overflow | Undefined behavior: the optimizer may delete your checks | Defined: panics in debug builds, wraps in release, or explicit (`checked_add`, `saturating_add`) |
| Ignored error code | Continues with a garbage value | Functions return `Result`; you cannot get the value without handling the error |
| Pointer to a local after return | Reads memory now used by other code | Compile error |
| Using `malloc` in "no heap" code | Allowed, banned only by coding rules | Compile error unless an allocator is set up |

## Panics

In C, writing past the end of an array is not detected. The write lands on whatever comes next in RAM, and the program keeps running with corrupted data:

```c
uint8_t buf[4];
buf[i] = 1;   // i == 4: overwrites the next variable, no error
```

A **panic** is Rust's reaction to a bug detected while the program runs. The same write checks the index first and, if it is out of range, stops the normal program and calls the **panic handler** instead:

```rust
let mut buf = [0u8; 4];
buf[i] = 1;   // i == 4: panic "index out of bounds: the len is 4 but the index is 4", with file and line
```

Every bare-metal program must choose its panic handler. A ready-made crate is usually used, or you write your own:

- **`panic-halt`**: loops forever. The device freezes in a known place, where a debugger can inspect it.
- **`panic-reset`**: restarts the chip, like pressing its reset button. The device recovers, but loses its current state.
- **`panic-probe`**: prints the message, file and line on the developer's PC through the debug probe. For development.

A custom handler is a few lines. This one restarts the chip, like `panic-reset`:

```rust
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {    // `!`: never returns
    cortex_m::peripheral::SCB::sys_reset();   // restart the chip
}
```

Production code usually avoids panics, e.g. `buf.get(i)` instead of `buf[i]`: `buf[i]` panics when out of range, while `buf.get(i)` returns an `Option` (the value, or `None`) that the caller must check.

## What Rust does not catch

- **Stack overflow**: same as C. A tool (`flip-link`) places the stack so an overflow causes an immediate fault instead of corruption.
- **Logic bugs**: wrong formulas, wrong protocols, wrong timing.

## `unsafe`

Some operations cannot be checked by the compiler, because their correctness depends on the hardware or on other code. Rust allows them only inside an **`unsafe`** block (`unsafe { ... }`), where the programmer, not the compiler, promises they are correct. On bare-metal these are mainly:

- **Reading or writing through a raw address** (a raw pointer), e.g. a hardware register.
- **Calling a function marked `unsafe`**, which includes every C function.
- **Accessing a changeable global** (`static mut`), e.g. one shared with an interrupt.

`unsafe` does not turn the other checks off: types, ownership and bounds checks still apply inside the block.

The standard pattern: a small `unsafe` core inside a safe function.

```rust
impl Pa5<Output> {
    pub fn set_high(&mut self) {
        // SAFETY: 0x4002_0018 is GPIOA's BSRR register. Writing bit 5 sets only PA5,
        // and we own the only Pa5 value.
        unsafe { (0x4002_0018 as *mut u32).write_volatile(1 << 5) };
    }
}
```

Application code calls `led.set_high()`, which is safe. In a typical firmware, `unsafe` lives in libraries (the PAC, the HAL, the startup code), and application modules can forbid it entirely with `#![forbid(unsafe_code)]`.

Why it matters: **undefined behavior** can come from any line of C, but only from `unsafe` blocks in Rust. When memory is corrupted, the search is limited to those blocks, and reviews can focus there. The honest limit: a bug inside `unsafe` code, including in a library, can still corrupt memory.

## C vs Rust at a glance

| | C | Rust |
|---|---|---|
| Memory bugs | Silent corruption | Compile errors or panics with location |
| Errors | Codes, easy to ignore | `Result`, must be handled |
| Fatal error handling | Project-specific | Required panic handler |
| Stack overflow | Silent | Silent, unless `flip-link` |
| Where undefined behavior can come from | Anywhere | Only `unsafe` blocks |
