# Hardware Access and Peripheral Ownership

## Bare-metal in short

A **bare-metal** program runs directly on a microcontroller, with no operating system. It starts at power-on and never exits. The chip has a CPU, **flash** (the program), **RAM** (variables), and **peripherals**: hardware blocks for pins (GPIO), serial buses (UART, SPI, I2C), timers, and so on.

You control a peripheral through its **registers**: 32-bit locations at fixed addresses. Writing to address `0x4002_0018` does not store anything in RAM; it changes the voltage on a pin. Register accesses must be **volatile**, which tells the compiler to perform every read and write exactly as written, because the hardware reacts to them.

Examples use the **ST Nucleo-F401RE** board (STM32F401, ARM Cortex-M4). Its LED is on pin `PA5`.

## Registers

Each register is split into **bit fields**. For example, `GPIOA_MODER` has 2 bits per pin: `00` = input, `01` = output, `10` = alternate function, `11` = analog.

### Where register descriptions come from

The **vendor** is the company that makes the chip: ST (STM32), Nordic (nRF), NXP, Microchip, Raspberry Pi (RP2040), Espressif (ESP32) and so on. Each publishes a reference manual describing every register.

**CMSIS** (Common Microcontroller Software Interface Standard) is Arm's standard for Cortex-M software. Arm writes the parts common to every Cortex-M core (e.g. `core_cm4.h`); each vendor writes the parts for its own chips. Two of these parts matter here:

- **Device headers** (e.g. `stm32f401xe.h`): C structs and macros for every register. This is what C firmware uses.
- **SVD files** (System View Description, e.g. `STM32F401.svd`): the same information as machine-readable XML: every peripheral, register, field, and the meaning of each value. Debuggers use it to show registers by name; Rust tools use it to generate code. Espressif also publishes SVD files for its RISC-V and Xtensa chips, so the format is not limited to Arm.

Both ship in the vendor's **CMSIS-Pack** (Device Family Pack), downloadable from [Arm's pack index](https://www.keil.arm.com/packs/), and usually also on the vendor's site or GitHub (e.g. inside ST's STM32CubeF4). The Arm-written parts are open source at [ARM-software/CMSIS_6](https://github.com/ARM-software/CMSIS_6).

### In C

The CMSIS device header gives each register a name and each bit a macro:

```c
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;                       // turn on port A's clock
GPIOA->MODER = (GPIOA->MODER & ~GPIO_MODER_MODER5)
             | GPIO_MODER_MODER5_0;                        // PA5 = output (01)
```

The macros are plain integers, so the compiler accepts mistakes like these:

```c
GPIOA->ODR = RCC_AHB1ENR_GPIOAEN;    // a mask that belongs to a different register
GPIOA->MODER |= 3u << 10;            // meant output (01), wrote analog (11)
```

### In Rust

There are several ways to reach a register. The first two are part of the language; the rest are ordinary crates (libraries).

| Option | What it is | Part of Rust? |
|---|---|---|
| Raw volatile access | `read_volatile` / `write_volatile` on an address, like a C `volatile` pointer. `unsafe`. | Yes (`core::ptr`) |
| Inline assembly | `asm!`, for CPU registers and instructions that are not memory-mapped (e.g. disabling interrupts). `unsafe`. | Yes (`core::arch`) |
| **PAC** | A typed register library for one chip, generated from its SVD file. | No |
| **HAL crate** | Drivers (GPIO, UART, ...) for one chip family, built on its PAC. | No |

#### Raw volatile access

The address and masks are plain integers, so this looks like C and allows the same mistakes:

```rust
const GPIOA_MODER: *mut u32 = 0x4002_0000 as *mut u32;
unsafe {
    let v = GPIOA_MODER.read_volatile();
    GPIOA_MODER.write_volatile((v & !(0b11 << 10)) | (0b01 << 10));   // PA5 = output (01)
}
```

#### Inline assembly

Rarely written by hand: the `cortex-m` and `riscv` crates wrap the common cases (e.g. `cortex_m::interrupt::disable()` emits `cpsid i`).

#### PAC (Peripheral Access Crate)

A PAC is the Rust counterpart of the CMSIS device header: one crate per chip, with a definition for every peripheral, register and field. The difference is that registers, fields and values are **types and methods** instead of integers, so the compiler can tell them apart. Nobody writes these by hand; they are generated from the vendor's SVD file.

With the `stm32f4` PAC, the C example becomes:

```rust
dp.RCC.ahb1enr().modify(|_, w| w.gpioaen().enabled());   // turn on port A's clock
dp.GPIOA.moder().modify(|_, w| w.moder5().output());     // PA5 = output
```

`modify` does a volatile read-modify-write; inside it you name only the fields you change. Every field is a method that exists only on its own register, and values have names (`output()`) instead of bit patterns. The first C mistake is a compile error (no `gpioaen` field on `ODR`), and the second would require writing `analog()` explicitly. Raw bits can still be written through `unsafe { w.bits(..) }`. The generated code is the same load, mask and store as the C version.

How a PAC is made:

1. Take the vendor's SVD file.
2. Fix its errors, which are common (e.g. stm32-rs applies YAML patches with `svdtools`).
3. Run **svd2rust**, which turns the SVD file into Rust source.
4. Publish the result on crates.io (`stm32f4`, `nrf52840-pac`, `rp2040-pac`, ...). Users add it as a dependency and never run svd2rust themselves.

Chips without an SVD file need their registers written by hand with one of the other options.

**Is it standard?** svd2rust is the de facto standard in Rust, maintained by the Rust Embedded Working Group and used by most PACs. It is not a vendor standard like CMSIS: apart from Espressif, vendors do not ship Rust PACs; the community does.

## Ownership: who may touch the hardware

In C, registers are global: any file can reconfigure any peripheral at any time. Two drivers configuring the same pin differently is found on the bench, if at all.

Rust's **ownership** rules (see [the Rust book, ch. 4](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)) say every value has one owner, and passing it somewhere **moves** it: the old name can no longer be used. svd2rust PACs use this: each peripheral is a value you can get only once, and a driver takes it over:

```rust
let dp = pac::Peripherals::take().unwrap();   // PAC: succeeds only the first time

let uart = Uart::new(dp.USART2, ...);         // USART2 moves into the driver
let other = Other::new(dp.USART2, ...);       // compile error: value already moved
```

These values are **zero bytes** in size; the address is part of the type. The tracking happens in the compiler and costs nothing at runtime. It holds for code that goes through the PAC or HAL; raw volatile access can still write any register ([chapter 3](3-memory-safety.md)).

## HAL crates

**HAL** (Hardware Abstraction Layer) is a general concept: a layer of driver functions (`gpio_set`, `uart_write`, ...) over the raw registers. ST's STM32Cube HAL is the usual one in C.

In Rust there is no single HAL. Each chip family has its own HAL crate, built on that family's PAC:

| Chip family | HAL crate | Maintained by |
|---|---|---|
| STM32 | `stm32f4xx-hal` (one per series), or `embassy-stm32` | Community (stm32-rs, Embassy) |
| Nordic nRF | `nrf-hal`, or `embassy-nrf` | Community |
| RP2040 / RP2350 | `rp2040-hal` / `rp235x-hal`, or `embassy-rp` | Community |
| ESP32 | `esp-hal` | Espressif |

The examples below use `stm32f4xx-hal`.

### Typestate: a pin's mode is part of its type

In C, the compiler doesn't know a pin's mode, so toggling (flipping high/low) a pin you never set as output still compiles:

```c
__HAL_RCC_GPIOA_CLK_ENABLE();
GPIO_InitTypeDef init = { .Pin = GPIO_PIN_5, .Mode = GPIO_MODE_OUTPUT_PP };
HAL_GPIO_Init(GPIOA, &init);
HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);   // compiles even without the Init lines
```

**Typestate** is a pattern where an object's state is stored in its type. In `stm32f4xx-hal`, a pin is `PA5<Input>` or `PA5<Output>` (input: the pin reads a voltage, e.g. a button; output: the pin drives a voltage, e.g. an LED), and only output pins have `toggle()`:

```rust
let pin = dp.GPIOA.split().pa5;              // PA5<Input>
pin.toggle();                                // compile error
let mut led = pin.into_push_pull_output();   // PA5<Output>
led.toggle();                                // OK
```

The mistake becomes a compile error, at no runtime cost. Embassy and `esp-hal` do the same with an `Output` driver: `Output::new(p.PA5, ...)`.

## Limits

- These checks come from the libraries, not the language. Raw volatile access has none of them.
- The PAC checks names and values, not order: forgetting a step still compiles. HAL crates catch more.
- HAL coverage varies by chip family.
- Typed APIs are verbose, and compiler errors for generic code take time to get used to.

## C vs Rust at a glance

| | C | Rust |
|---|---|---|
| Wrong register / field | Compiles | Compile error (PAC); compiles (raw volatile) |
| Two drivers on one peripheral | Found at runtime | Compile error |
| Pin used in the wrong mode | Silently nothing happens | Compile error |
| Runtime cost | — | Same instructions as C |
