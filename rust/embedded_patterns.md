# Embedded Rust & no_std Patterns

Patterns for writing Rust on microcontrollers and resource-constrained targets. Covers no_std fundamentals, embedded-hal, typestate peripherals, Embassy async, and ESP32/RP2040 specifics. Continues from Skills 1-76.

---

## Skill 77: no_std Fundamentals — Shedding the Standard Library

**Pattern**: Use `#![no_std]` to target bare-metal, relying on `core` (always available) and optionally `alloc` (if you have a global allocator).

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

// Bare-metal entry point (cortex-m-rt provides the reset vector)
#[cortex_m_rt::entry]
fn main() -> ! {
    // Your code here
    loop {}
}

// Required: define panic behavior
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**What you lose without `std`**:
- `println!`, `format!` → use `core::fmt::Write` or `defmt`
- `Vec`, `String`, `HashMap` → use `heapless` crate or fixed-size arrays
- `std::io` → use `embedded-io` traits
- `std::time` → use hardware timers or `embassy-time`
- Threads → use interrupts or async tasks

**What you keep from `core`**:
- `Option`, `Result`, iterators, `core::fmt`
- All traits (`Clone`, `Debug`, `From`, etc.)
- Slices, arrays, primitive types
- `core::cell`, `core::sync::atomic`

**Skill to practice**: Take any small utility you've written and try to make it `#![no_std]` compatible. Replace any `std::` imports with `core::` equivalents.

---

## Skill 78: The embedded-hal Trait Ecosystem

**Pattern**: `embedded-hal` defines hardware-agnostic traits so drivers work across all chips. Write your drivers against traits, not concrete HALs.

```rust
use embedded_hal::digital::OutputPin;
use embedded_hal::spi::SpiDevice;

// This driver works on ANY chip that implements the traits
pub struct MySensor<SPI, CS> {
    spi: SPI,
    cs: CS,
}

impl<SPI, CS, E> MySensor<SPI, CS>
where
    SPI: SpiDevice<Error = E>,
    CS: OutputPin,
{
    pub fn new(spi: SPI, cs: CS) -> Self {
        Self { spi, cs }
    }

    pub fn read_value(&mut self) -> Result<u16, E> {
        let mut buf = [0u8; 2];
        self.spi.read(&mut buf)?;
        Ok(u16::from_be_bytes(buf))
    }
}
```

**Key embedded-hal traits**:
| Trait | Purpose |
|-------|---------|
| `OutputPin` / `InputPin` | GPIO control |
| `SpiDevice` / `SpiBus` | SPI communication |
| `I2c` | I2C communication |
| `Pwm` | PWM output |
| `DelayNs` | Blocking delay |
| `serial::Read/Write` | UART |

**embedded-hal 1.0 vs 0.2**: The 1.0 release (stable) uses `&mut self` consistently, splits `SpiBus` (raw bus) from `SpiDevice` (bus + chip select management), and uses associated error types throughout.

**Skill to practice**: Write a simple driver (e.g., for an LED or temperature sensor) using only `embedded-hal` traits, then test it on two different boards.

---

## Skill 79: Typestate GPIO — Compile-Time Pin Configuration

**Pattern**: Use the type system to encode pin states (input/output/alternate function) so misconfiguration is a compile error.

```rust
// HALs encode pin mode in the type
use hal::gpio::{Pin, Input, Output, PushPull, Floating};

// Can only read from an input pin
fn read_button(pin: &Pin<Input<Floating>>) -> bool {
    pin.is_high()
}

// Can only write to an output pin
fn set_led(pin: &mut Pin<Output<PushPull>>, on: bool) {
    if on { pin.set_high(); } else { pin.set_low(); }
}

// Transitioning consumes the old state, returns new state
let input_pin: Pin<Input<Floating>> = gpioa.pa0.into_floating_input();
let output_pin: Pin<Output<PushPull>> = input_pin.into_push_pull_output();
// input_pin is now consumed — can't accidentally use it
```

**Why this matters**: On microcontrollers, configuring a pin wrong can damage hardware. Typestate catches this at compile time instead of runtime.

**The pattern generalizes**: Any peripheral with distinct modes (UART configured vs unconfigured, SPI with/without chip select, timer running vs stopped) can use typestate.

**Skill to practice**: When wrapping a peripheral, encode its state machine in types. If transitioning from state A to B requires configuration, make it a method that consumes `Self<A>` and returns `Self<B>`.

---

## Skill 80: Heapless Collections — Fixed-Capacity Data Structures

**Pattern**: Use `heapless` crate for `Vec`, `String`, `Queue`, and `Map` with compile-time capacity limits — no allocator needed.

```rust
use heapless::Vec;
use heapless::String;

// Vec with capacity 16, stored entirely on the stack
let mut buffer: Vec<u8, 16> = Vec::new();
buffer.push(42).unwrap(); // Returns Err if full
buffer.extend_from_slice(&[1, 2, 3]).unwrap();

// Fixed-capacity string
let mut name: String<32> = String::new();
core::fmt::write(&mut name, format_args!("sensor_{}", 1)).unwrap();

// MPSC queue for interrupt → main communication
use heapless::mpmc::MpMcQueue;
static QUEUE: MpMcQueue<Event, 8> = MpMcQueue::new();

// In interrupt handler:
QUEUE.enqueue(Event::ButtonPressed).ok();

// In main loop:
if let Some(event) = QUEUE.dequeue() {
    handle(event);
}
```

**Key types**:
| Type | Use Case |
|------|----------|
| `Vec<T, N>` | Fixed-capacity growable array |
| `String<N>` | Fixed-capacity UTF-8 string |
| `LinearMap<K, V, N>` | Small map (linear search, good for N < 16) |
| `FnvIndexMap<K, V, N>` | Hash map with fixed capacity |
| `spsc::Queue<T, N>` | Single-producer single-consumer lock-free queue |
| `mpmc::MpMcQueue<T, N>` | Multi-producer multi-consumer queue |
| `pool::Pool` | Memory pool for dynamic-ish allocation |

**Skill to practice**: Replace a `std::vec::Vec` with `heapless::Vec` in a data processing function. Handle the capacity-full case explicitly.

---

## Skill 81: Memory Layout & Linker Scripts

**Pattern**: Embedded targets need explicit memory layout. The linker script (`memory.x`) tells the linker where RAM and Flash are.

```
/* memory.x — typical Cortex-M layout */
MEMORY
{
    FLASH : ORIGIN = 0x08000000, LENGTH = 256K
    RAM   : ORIGIN = 0x20000000, LENGTH = 64K
}
```

**Key sections**:
- `.text` → executable code (in FLASH)
- `.rodata` → constants, string literals (in FLASH)
- `.data` → initialized statics (stored in FLASH, copied to RAM at boot)
- `.bss` → zero-initialized statics (in RAM)
- `.stack` → stack (top of RAM, grows down)

**Boot sequence** (cortex-m-rt):
1. Reset handler runs
2. `.data` copied from FLASH to RAM
3. `.bss` zeroed
4. `main()` called

**`#[link_section]` for special placement**:
```rust
// Place a buffer in a specific RAM region (e.g., DMA-capable memory)
#[link_section = ".dma_buffer"]
static mut DMA_BUF: [u8; 256] = [0; 256];
```

**Skill to practice**: Read the `memory.x` for your target board. Understand where your code and data live. Use `cargo size` or `cargo bloat` to see how much Flash/RAM you're using.

---

## Skill 82: Interrupt Handling — Safe Patterns for Shared State

**Pattern**: Interrupts preempt your code at any time. Use `critical-section`, atomics, or lock-free queues to safely share state.

```rust
use core::cell::RefCell;
use critical_section::Mutex;

// Global shared state — wrapped in Mutex<RefCell<Option<T>>>
static BUTTON_PIN: Mutex<RefCell<Option<ButtonPin>>> = Mutex::new(RefCell::new(None));
static PRESS_COUNT: core::sync::atomic::AtomicU32 = AtomicU32::new(0);

// Initialize in main (move peripheral into global)
critical_section::with(|cs| {
    BUTTON_PIN.borrow_ref_mut(cs).replace(button_pin);
});

// Interrupt handler
#[interrupt]
fn EXTI0() {
    // For simple counters, atomics are best
    PRESS_COUNT.fetch_add(1, Ordering::Relaxed);

    // For peripheral access, use critical section
    critical_section::with(|cs| {
        if let Some(ref mut pin) = *BUTTON_PIN.borrow_ref_mut(cs) {
            pin.clear_interrupt_pending();
        }
    });
}
```

**Hierarchy of approaches** (simplest → most complex):
1. **Atomics** — for counters, flags, simple values
2. **critical-section Mutex** — for peripheral handles, complex state
3. **heapless queues** — for passing data from ISR to main loop
4. **Embassy** — avoid raw interrupts entirely with async

**Anti-pattern**: Never use `static mut` directly — it's instant UB if an interrupt fires during access. Always wrap in a synchronization primitive.

**Skill to practice**: Write a button-press counter using atomics, then refactor to pass events through a `heapless::spsc::Queue`.

---

## Skill 83: Singleton Peripherals & PAC/HAL Split

**Pattern**: Rust's ownership system enforces that each peripheral is used by only one part of the code. The PAC (Peripheral Access Crate) provides raw register access; the HAL wraps it safely.

```rust
// PAC: auto-generated from SVD, raw register access
let dp = pac::Peripherals::take().unwrap(); // Returns Option — only works once!

// HAL: wraps PAC into safe, ergonomic types
let rcc = dp.RCC.constrain();
let clocks = rcc.cfgr.sysclk(72.MHz()).freeze();
let gpioa = dp.GPIOA.split(); // Splits port into individual pins
let mut led = gpioa.pa5.into_push_pull_output();
```

**The layers**:
```
Your code
  ↓ uses embedded-hal traits
HAL crate (e.g., stm32f4xx-hal, esp-hal)
  ↓ wraps registers safely
PAC crate (e.g., stm32f4, esp32c3)
  ↓ auto-generated from SVD/register descriptions
Hardware registers
```

**`Peripherals::take()` pattern**: Returns `Some(peripherals)` on first call, `None` thereafter. This is a runtime singleton — ensures only one owner of the hardware. The underlying implementation uses an `AtomicBool`.

**Skill to practice**: Trace the path from `Peripherals::take()` through the HAL to understand how your chip's registers get wrapped. Read the PAC source for one peripheral.

---

## Skill 84: defmt — Efficient Embedded Logging

**Pattern**: Use `defmt` instead of `core::fmt` for logging on embedded. It sends format strings at compile time and only transmits minimal data at runtime.

```rust
use defmt::*;

#[defmt::panic_handler]
fn panic() -> ! { loop {} }

fn read_sensor(value: u16) {
    info!("Sensor reading: {}", value);        // costs ~2 bytes over the wire
    debug!("Raw bytes: {:x}", value);
    warn!("Temperature high: {} °C", value);

    // Structured data
    #[derive(defmt::Format)]
    struct Reading { temp: i16, humidity: u8 }
    let r = Reading { temp: 25, humidity: 60 };
    info!("Reading: {:?}", r);
}
```

**Why defmt over println/log**:
- `core::fmt` pulls in ~10-20KB of formatting code
- `defmt` sends only a format string index + raw data
- Decoding happens on the host (via `probe-run` or `defmt-rtt`)
- Log levels can be filtered at compile time (zero cost when disabled)

**Transport options**: RTT (Real-Time Transfer, via debug probe), UART, ITM

**Skill to practice**: Set up `defmt` + `probe-run` on a project. Compare binary size with and without `defmt` vs `core::fmt::write`.

---

## Skill 85: Embassy Fundamentals — Async on Bare Metal

**Pattern**: Embassy brings `async/await` to embedded Rust. Each task is a lightweight state machine — no OS threads, no heap allocation needed.

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    // Spawn concurrent tasks — each is a zero-alloc state machine
    spawner.spawn(blink(p.PA5.into())).unwrap();
    spawner.spawn(sensor_loop()).unwrap();

    // Main task can also do async work
    loop {
        info!("Main loop tick");
        Timer::after(Duration::from_secs(5)).await;
    }
}

#[embassy_executor::task]
async fn blink(mut led: Output<'static>) {
    loop {
        led.toggle();
        Timer::after(Duration::from_millis(500)).await;
    }
}

#[embassy_executor::task]
async fn sensor_loop() {
    loop {
        let value = read_sensor().await;
        info!("Sensor: {}", value);
        Timer::after(Duration::from_secs(1)).await;
    }
}
```

**Key concepts**:
- **No OS needed**: Embassy's executor runs tasks cooperatively on bare metal
- **Tasks are `'static`**: They own their data (no borrowing from `main`)
- **`Timer::after().await`** replaces blocking delays — other tasks run while waiting
- **Interrupts drive the executor**: When hardware is ready, the corresponding Waker is signaled

**Embassy vs RTIC**: Embassy is fully async (cooperative), RTIC is interrupt-driven (preemptive). Embassy is simpler for I/O-heavy code; RTIC is better for hard real-time with strict priority requirements.

**Skill to practice**: Convert a main-loop-with-delays program into Embassy tasks. Notice how each logical concern becomes its own task.

---

## Skill 86: Embassy HAL Integration — Async Peripherals

**Pattern**: Embassy HALs make peripheral operations async. SPI transfers, UART reads, and I2C transactions `await` completion instead of busy-waiting.

```rust
use embassy_stm32::spi::{self, Spi};
use embassy_stm32::time::Hertz;

#[embassy_executor::task]
async fn spi_task(mut spi: Spi<'static, spi::Async>) {
    let mut tx_buf = [0x01, 0x02, 0x03];
    let mut rx_buf = [0u8; 3];

    // This awaits DMA completion — CPU is free for other tasks
    spi.transfer(&mut rx_buf, &tx_buf).await.unwrap();
    info!("SPI received: {:x}", rx_buf);
}

// UART echo with async read/write
use embassy_stm32::usart::{Uart, UartRx, UartTx};

#[embassy_executor::task]
async fn uart_echo(mut rx: UartRx<'static, embassy_stm32::mode::Async>,
                   mut tx: UartTx<'static, embassy_stm32::mode::Async>) {
    let mut buf = [0u8; 64];
    loop {
        let n = rx.read_until_idle(&mut buf).await.unwrap();
        tx.write(&buf[..n]).await.unwrap();
    }
}
```

**DMA under the hood**: Embassy HALs use DMA automatically for async operations. The CPU sleeps (WFI) while DMA transfers data, then the DMA-complete interrupt wakes the task.

**Splitting peripherals**: Many peripherals can be split into independent halves:
```rust
let (tx, rx) = uart.split();
spawner.spawn(reader(rx)).unwrap();
spawner.spawn(writer(tx)).unwrap();
```

**Skill to practice**: Write an async SPI driver that reads a sensor periodically. Compare power consumption vs a busy-wait approach.

---

## Skill 87: Embassy Signals, Channels & Synchronization

**Pattern**: Use Embassy's synchronization primitives to communicate between tasks without shared mutable state.

```rust
use embassy_sync::signal::Signal;
use embassy_sync::channel::Channel;
use embassy_sync::mutex::Mutex;
use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;

// Signal: latest-value, lossy (overwrites if not read)
static BUTTON_SIGNAL: Signal<CriticalSectionRawMutex, ButtonEvent> = Signal::new();

// Channel: buffered, lossless MPMC queue
static EVENT_CHANNEL: Channel<CriticalSectionRawMutex, Event, 8> = Channel::new();

// Mutex: for shared resource access
static DISPLAY: Mutex<CriticalSectionRawMutex, RefCell<Option<Display>>> =
    Mutex::new(RefCell::new(None));

#[embassy_executor::task]
async fn button_handler() {
    loop {
        let event = BUTTON_SIGNAL.wait().await; // Blocks until signaled
        EVENT_CHANNEL.send(Event::Button(event)).await;
    }
}

#[embassy_executor::task]
async fn event_processor() {
    loop {
        let event = EVENT_CHANNEL.receive().await;
        match event {
            Event::Button(e) => handle_button(e),
            Event::Sensor(v) => handle_sensor(v),
        }
    }
}
```

**Choosing the right primitive**:
| Primitive | Buffered | Lossy | Use Case |
|-----------|----------|-------|----------|
| `Signal` | No (1 slot) | Yes (overwrites) | Latest sensor reading, button press |
| `Channel` | Yes (N slots) | No (backpressure) | Event queues, command streams |
| `Mutex` | N/A | N/A | Shared resource (display, storage) |
| `PubSubChannel` | Yes | Configurable | Multiple subscribers to same events |

**RawMutex types**: `CriticalSectionRawMutex` (single-core), `ThreadModeRawMutex` (main thread only), `NoopRawMutex` (single-task only, zero cost).

**Skill to practice**: Build a producer-consumer pattern with a `Channel` between a sensor-reading task and a display-updating task.

---

## Skill 88: Power Management & Sleep in Embassy

**Pattern**: Embassy automatically puts the CPU to sleep (WFI) when no tasks are ready. Design your tasks around `await` points to maximize sleep time.

```rust
// BAD: busy-waiting wastes power
loop {
    if sensor.data_ready() {
        let v = sensor.read();
        process(v);
    }
    // CPU is at 100% even when idle
}

// GOOD: async waiting, CPU sleeps between events
loop {
    sensor.wait_for_data_ready().await;  // CPU sleeps here
    let v = sensor.read_async().await;    // CPU sleeps during DMA
    process(v);
}

// Use Timer for periodic wakeups
use embassy_time::{Timer, Duration, Ticker};

#[embassy_executor::task]
async fn periodic_sensor() {
    let mut ticker = Ticker::every(Duration::from_secs(1));
    loop {
        ticker.next().await;  // Precise periodic wakeup
        let reading = read_sensor().await;
        publish_reading(reading);
    }
}
```

**Ticker vs Timer**:
- `Timer::after()` — delay *from now*, can drift
- `Ticker::every()` — fixed-interval, compensates for execution time

**Power tips**:
1. Use `await` everywhere possible — every await is a sleep opportunity
2. Use `Ticker` for periodic tasks — it compensates for drift
3. Split slow peripherals (UART, I2C) into async tasks — don't block the executor
4. Use `select!` to wake on whichever event comes first

**Skill to practice**: Measure current consumption of a polling loop vs an Embassy async loop doing the same work. Use a Ticker for periodic sensor reads.

---

## Skill 89: ESP32-Specific Patterns with esp-hal

**Pattern**: ESP32 chips use `esp-hal` (bare-metal) or `esp-idf-hal` (FreeRTOS-based). For no_std Embassy, use `esp-hal` + `esp-wifi`.

```rust
#![no_std]
#![no_main]

use esp_hal::prelude::*;
use esp_hal::gpio::{Io, Level, Output};
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};

#[esp_hal_embassy::main]
async fn main(spawner: Spawner) {
    let peripherals = esp_hal::init(esp_hal::Config::default());
    let io = Io::new(peripherals.GPIO, peripherals.IO_MUX);

    let led = Output::new(io.pins.gpio2, Level::Low);
    spawner.spawn(blink(led)).unwrap();
}

#[embassy_executor::task]
async fn blink(mut led: Output<'static>) {
    loop {
        led.toggle();
        Timer::after(Duration::from_millis(500)).await;
    }
}
```

**ESP32 ecosystem choices**:
| Approach | Crate | Allocator | Use Case |
|----------|-------|-----------|----------|
| Bare-metal no_std | `esp-hal` | None/`esp-alloc` | Maximum control, smallest binary |
| Embassy no_std | `esp-hal` + `esp-hal-embassy` | None/`esp-alloc` | Async bare-metal |
| std (FreeRTOS) | `esp-idf-hal` | Yes (libc) | WiFi/BLE easiest, larger binary |

**WiFi on no_std**: `esp-wifi` provides async WiFi/BLE without `std`. It requires an allocator (`esp-alloc`) for internal buffers.

```rust
use esp_wifi::wifi::{WifiStaDevice, WifiController};
// WiFi setup requires esp-alloc initialized
esp_alloc::heap_allocator!(size: 72 * 1024);
```

**Skill to practice**: Get an ESP32 blinking with `esp-hal` + Embassy. Then try adding WiFi with `esp-wifi` to see the no_std networking stack.

---

## Skill 90: Raspberry Pi Pico / RP2040 with Embassy

**Pattern**: The RP2040 has excellent Embassy support via `embassy-rp`. It's a great platform for learning Embassy.

```rust
#![no_std]
#![no_main]

use embassy_rp::gpio::{Level, Output};
use embassy_rp::peripherals::PIN_25;
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_rp::init(Default::default());
    spawner.spawn(blink(p.PIN_25)).unwrap();
}

#[embassy_executor::task]
async fn blink(pin: PIN_25) {
    let mut led = Output::new(pin, Level::Low);
    loop {
        led.toggle();
        Timer::after(Duration::from_millis(200)).await;
    }
}
```

**RP2040 special features**:
- **PIO (Programmable I/O)**: Embassy wraps PIO state machines for custom protocols
- **Dual-core**: Embassy can run tasks on both cores
- **USB**: `embassy-usb` provides async USB device support
- **Flash**: `embassy-rp` includes async flash read/write

**Skill to practice**: Build a USB HID device (keyboard/mouse) using `embassy-usb` on the RP2040 — it demonstrates async USB, PIO, and multi-task coordination.

---

## Skill 91: Testing Embedded Code — Host-Side & On-Target

**Pattern**: Test business logic on the host (fast iteration), test hardware interaction on-target (accuracy).

```rust
// Business logic: pure functions, testable on host
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_checksum() {
        assert_eq!(calculate_checksum(&[0x01, 0x02, 0x03]), 0x06);
    }

    #[test]
    fn test_state_machine() {
        let mut sm = ProtocolStateMachine::new();
        sm.feed(0x55); // sync byte
        sm.feed(0x03); // length
        assert_eq!(sm.state(), State::ReadingPayload);
    }
}

// Hardware-dependent: use embedded-hal-mock for unit tests
#[cfg(test)]
mod hal_tests {
    use embedded_hal_mock::eh1::spi::{Mock, Transaction};

    #[test]
    fn test_sensor_driver() {
        let expectations = [
            Transaction::transfer_in_place(vec![0x00, 0x00], vec![0x01, 0x42]),
        ];
        let mut spi = Mock::new(&expectations);
        let mut sensor = MySensor::new(&mut spi);
        assert_eq!(sensor.read_value().unwrap(), 0x0142);
        spi.done(); // verify all expectations met
    }
}
```

**Testing strategy**:
1. **Separate logic from hardware**: Keep parsing, state machines, calculations in pure functions
2. **`embedded-hal-mock`**: Provides mock implementations of all embedded-hal traits
3. **`defmt-test`**: Run tests on-target with defmt output
4. **`probe-run`**: Flash and run tests on real hardware

**Skill to practice**: Extract the protocol parsing from a driver into a pure function and write host-side tests for it. Then add `embedded-hal-mock` tests for the SPI/I2C layer.

---

## Skill 92: Embedded Anti-Patterns to Avoid

**Pattern**: Common mistakes in embedded Rust and their corrections.

**1. Blocking in async context**:
```rust
// BAD: blocks the entire executor
#[embassy_executor::task]
async fn bad_task() {
    loop {
        cortex_m::asm::delay(1_000_000); // blocks all tasks!
        do_work();
    }
}

// GOOD: use async delay
#[embassy_executor::task]
async fn good_task() {
    loop {
        Timer::after(Duration::from_millis(100)).await; // yields to executor
        do_work();
    }
}
```

**2. Unbounded `static mut` access**:
```rust
// BAD: data race between main and interrupt
static mut COUNTER: u32 = 0;
#[interrupt]
fn TIMER() { unsafe { COUNTER += 1; } } // UB!

// GOOD: use atomic
static COUNTER: AtomicU32 = AtomicU32::new(0);
#[interrupt]
fn TIMER() { COUNTER.fetch_add(1, Ordering::Relaxed); }
```

**3. Ignoring peripheral ownership**:
```rust
// BAD: cloning a peripheral reference
let uart1 = get_uart(); // hypothetical unsafe global access
let uart2 = get_uart(); // two mutable refs to same peripheral — UB!

// GOOD: take() + move into tasks
let uart = dp.USART1.take().unwrap();
spawner.spawn(uart_task(uart)).unwrap(); // moved, single owner
```

**4. Allocating in no_std without thinking**:
```rust
// BAD: panics on OOM with no error handling
esp_alloc::heap_allocator!(size: 1024); // too small
let big = alloc::vec![0u8; 2048]; // panic!

// GOOD: use heapless or handle allocation failure
let mut buf: heapless::Vec<u8, 2048> = heapless::Vec::new();
if buf.extend_from_slice(&data).is_err() {
    defmt::warn!("Buffer full, dropping data");
}
```

**5. Forgetting to clear interrupt flags**:
```rust
// BAD: interrupt fires continuously
#[interrupt]
fn EXTI0() {
    do_work(); // never clears the pending flag → infinite loop
}

// GOOD: always clear the flag
#[interrupt]
fn EXTI0() {
    pac::EXTI.pr.write(|w| w.pr0().set_bit()); // clear pending
    do_work();
}
```

**Skill to practice**: Review one of your existing embedded projects for these anti-patterns. Fix any you find.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
| 77 | no_std Fundamentals | `core` always, `alloc` optional, define `#[panic_handler]` |
| 78 | embedded-hal Traits | Write drivers against traits, not concrete HALs |
| 79 | Typestate GPIO | Encode pin modes in types, transitions consume old state |
| 80 | Heapless Collections | `Vec<T, N>` — fixed capacity, no allocator needed |
| 81 | Memory Layout | Linker script defines FLASH/RAM, sections map to regions |
| 82 | Interrupt Safety | Atomics → critical_section → heapless queues → Embassy |
| 83 | Singleton Peripherals | `Peripherals::take()` → HAL → type-safe pins |
| 84 | defmt Logging | Compile-time format strings, minimal runtime cost |
| 85 | Embassy Fundamentals | Async tasks = zero-alloc state machines on bare metal |
| 86 | Embassy HAL | Peripheral ops are async, DMA-backed, CPU sleeps |
| 87 | Signals & Channels | `Signal` (latest value), `Channel` (buffered queue) |
| 88 | Power Management | Every `await` is a sleep opportunity; use `Ticker` |
| 89 | ESP32 Patterns | `esp-hal` + `esp-hal-embassy` for no_std async |
| 90 | RP2040 Patterns | `embassy-rp` with PIO, USB, dual-core support |
| 91 | Testing Embedded | Separate logic from hardware, use `embedded-hal-mock` |
| 92 | Anti-Patterns | No blocking in async, no `static mut`, clear IRQ flags |

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-16 | Initial version — Skills 77-92 from stdlib/ecosystem analysis |
