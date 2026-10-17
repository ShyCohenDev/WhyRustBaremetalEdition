# Interrupts and Shared State

## The problem

An **interrupt** lets a peripheral (e.g. a timer) pause the CPU: the CPU saves where it was, runs an **interrupt handler** function, then continues where it stopped. Saving the CPU registers is automatic (done by the hardware on Cortex-M); keeping shared variables consistent is not.

A handler can run between any two instructions of `main`. When both change the same variable, updates get lost:

```c
uint32_t counter;
void TIM2_IRQHandler(void) { counter++; }   // interrupt handler of timer 2
int main(void) { counter++; }                // compiles to 3 instructions: load, add, store
```

```text
main:    load counter   → 5
                              ── interrupt ── handler: counter = 6
main:    add 1          → 6
main:    store 6               (the handler's increment is lost)
```

Such a **race condition** can fail once a week and never on the developer's desk.

## Finding the handler

The CPU finds handlers through the **vector table** (a list of handler addresses at the start of flash). In C, the table comes from the vendor's startup file, and your handler is matched by name:

```c
void TIM2_IRQHandler(void) { ... }   // replaces the default handler
void TIM2_IRQhandler(void) { ... }   // typo: compiles, never called; the chip freezes in the default handler
```

In Rust, the PAC generates the vector table, and its `#[interrupt]` attribute checks the name against the chip's list of interrupts:

```rust
#[interrupt]
fn TIM2() { ... }   // compile error if the chip has no TIM2 interrupt
```

## Sharing a variable with a handler

In C, the fix is a **critical section** (interrupts turned off around the access, so nothing can run in between):

```c
__disable_irq();
counter++;
__enable_irq();   // wrong if interrupts were already off: this turns them back on
```

Nothing forces it. A shared variable looks like any other global, so a missing critical section compiles.

In Rust, a global that can change is a `static mut`, and every access to it needs `unsafe`. The race can still be written, but not by accident. Safe code uses one of two tools instead:

**Atomics** (`core::sync::atomic`, part of Rust), for single numbers. The CPU does load, add and store as one step that cannot be interrupted:

```rust
static COUNTER: AtomicU32 = AtomicU32::new(0);
COUNTER.fetch_add(1, Ordering::Relaxed);   // no critical section needed
```

Not every chip has these instructions (Cortex-M0 has no `fetch_add`).

**A critical-section mutex** (the `critical-section` crate, used by most HAL crates), for anything bigger. The data can only be reached with a token (`cs`) that exists only while interrupts are off:

```rust
static STATS: Mutex<RefCell<Stats>> = Mutex::new(RefCell::new(Stats::new()));

critical_section::with(|cs| {             // interrupts off; previous state restored after
    STATS.borrow_ref_mut(cs).count += 1;  // without `cs`: compile error
});
```

A forgotten critical section becomes a compile error, at the same cost as the C version. It is verbose.

## DMA

**DMA** (Direct Memory Access) is hardware that copies data between RAM and a peripheral without the CPU. While a transfer runs, the buffer belongs to the hardware, but in C nothing stops the code from changing or freeing it:

```c
void send_status(void) {
    uint8_t buf[64];                          // local: lives on the stack
    HAL_UART_Transmit_DMA(&huart2, buf, 64);  // starts the transfer and returns
}   // buf is gone, but the DMA keeps reading that memory. Compiles fine.
```

In HAL crates, starting a transfer **moves** the buffer in, and finishing gives it back (simplified; the exact API differs per HAL crate):

```rust
let transfer = Transfer::start(dma_channel, uart_tx, buffer);   // buffer moves in
buffer[0] = 1;                                                  // compile error: buffer was moved
let (dma_channel, uart_tx, buffer) = transfer.wait();           // buffer moves back
```

The buffer must also live for the whole program (`'static`), so the stack buffer above is rejected. Embassy gets the same result with `async`: the buffer is borrowed until the transfer's `.await` finishes.

## C vs Rust at a glance

| | C | Rust |
|---|---|---|
| Misspelled handler | Silently never called | Compile error |
| Global shared with a handler | Looks like any global | `unsafe` required for every access |
| Safe sharing | Discipline | Atomics, or a mutex needing a critical-section token |
| DMA buffer misuse | Compiles | Compile error |
