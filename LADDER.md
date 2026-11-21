# Embedded ladder: TM4C123 from bare metal to a bootloader

Board: EK-TM4C123GXL (TM4C123GH6PM, Cortex-M4F, 80 MHz max, 256 KB flash, 32 KB SRAM).
Toolchain: arm-none-eabi-gcc, make, OpenOCD (ICDI), arm-none-eabi-gdb.
Started: 21 Nov 2026. Pace: ~6 h/week through Stage 2, more from Stage 3 on.

## Rules

- **One codebase, growing layer by layer.** Not separate demos. Later rungs use the
  drivers from earlier rungs, so bugs come from how the layers interact.
- **No TivaWare driver calls, no vendor HAL.** Registers and the datasheet only.
  CMSIS / TI header files for register definitions are fine.
- **I write all the code.** Claude gives briefs, points to datasheet sections, answers
  with questions or the smallest hint that works, and reviews when asked.
- **Each rung closes with:** the done-test passing, a commit, a `LOG.md` entry, and the
  rung's *Check* questions answered out loud.
- **Habits:** write the expected output before running, write down what I don't know
  as comments in the code, and check results against a known-good reference (datasheet
  values, a scope or logic analyzer trace, `objdump`/`nm`/the `.map` file).

## Status key

`current` = the rung being worked on (only ever one) · `done` · `todo`

## Sources

- **DS**: TM4C123GH6PM datasheet (the primary source)
- **CM4**: ARM Cortex-M4 Devices Generic User Guide (NVIC, SysTick, SCB, PendSV, MPU, fault registers)
- **LP**: EK-TM4C123GXL LaunchPad user's guide (pins: LEDs, buttons, UART0 on the debug USB)
- **Yiu**: Joseph Yiu, *The Definitive Guide to ARM Cortex-M3 and Cortex-M4 Processors*
- **ld**: GNU ld manual, chapter "Linker Scripts"

---

## Stage 1: from bare metal to main

### Rung 1: Boot from nothing (`current`)
**Goal:** your own startup code, vector table and linker script; `Makefile`; flash it,
step through `Reset_Handler` in GDB, then blink an LED with raw register writes.
**Done-test:**
- `make` produces an ELF from your own startup file and `.ld`, with no vendor startup file.
- In GDB you break at `Reset_Handler`, step through the `.data` copy and the `.bss` zeroing,
  and reach `main`.
- An initialised global and a zero-initialised global both have the right values in `main`,
  checked in GDB.
- An LED blinks with a busy-wait delay.
- From `arm-none-eabi-nm`/`objdump -h` and the `.map` file, you can say where every
  section's LMA and VMA are.
**Read:** DS: Memory Map, Internal Memory (Flash/SRAM), System Control (RCGCGPIO),
GPIO. CM4: Exception model, vector table. LP: the user LED pins. ld: the
`MEMORY`, `SECTIONS` and `AT>` commands.
**Check:** what happens from power-on to `main()`? What are `.data` and `.bss`, and
where does each live in flash and in RAM?

### Rung 2: Clocks and time (`todo`)
**Goal:** PLL to 80 MHz; SysTick millisecond timebase; `delay_ms`; uptime counter.
**Done-test:** the LED toggles at 1.000 Hz, checked against a stopwatch over 60 s or
against a scope or logic analyzer; you can state the PLL configuration and why each
step is in that order.
**Read:** DS: System Control (clock tree, RCC/RCC2, PLL). CM4: SysTick.
**Check:** why does the clock tree matter for peripherals and power? What is the jitter
of your timebase?

### Rung 3: Debugging and faults (`todo`)
**Goal:** a solid GDB + OpenOCD workflow; trigger a HardFault on purpose and decode it;
a fault handler that dumps the stacked registers.
**Done-test:** three deliberate faults (bad address, unaligned or divide-by-zero with
the trap enabled, executing from a non-executable address), and for each one your
handler reports the stacked PC, LR, CFSR and HFSR, and you map the PC back to the
source line.
**Read:** CM4: Fault handling, SCB fault status registers (CFSR, HFSR, MMFAR, BFAR),
exception entry and stacking. Yiu: Fault exceptions chapter.
**Check:** how do you debug a HardFault?

## Stage 2: interrupts and concurrency

### Rung 4: GPIO interrupts and the NVIC (`todo`)
**Goal:** button edge interrupt with debounce; two interrupt sources at different
priorities; show the pending bit by masking, pressing, then unmasking.
**Done-test:** with interrupts masked, a button press sets the pending bit (seen in GDB)
and the handler runs once you unmask; a higher-priority interrupt preempts a
lower-priority handler (shown with GPIO toggles on the logic analyzer, or with
counters); bounce is gone.
**Read:** DS: GPIO (interrupt registers), Interrupts table. CM4: NVIC, priority
grouping, PRIMASK/BASEPRI.
**Check:** UART TX is slow: what do you do? What happens to an interrupt that arrives
while interrupts are disabled?

### Rung 5: UART, polled then interrupt-driven (`todo`)
**Goal:** UART0 at 115200 8N1; polled TX/RX first, then an ISR-driven SPSC ring
buffer and a `printf` on top.
**Done-test:** echo with no lost bytes when you paste a 4 KB block; you measure CPU time
blocked in TX before and after (GPIO toggle plus logic analyzer, or SysTick
counts) and log the numbers.
**Read:** DS: UART (baud-rate divisor, FIFOs, interrupts), GPIO alternate functions.
LP: UART0 over the debug USB.
**Check:** the slow-TX question from rung 4, answered with your own numbers.

### Rung 6: Race conditions on purpose (`todo`)
**Goal:** break the ring buffer (shared counter, non-atomic read-modify-write, missing
`volatile`), watch it fail, then fix it with critical sections, atomic access and
barriers.
**Done-test:** a stress setup that reproduces each bug reliably, and for each fix a
before/after run showing it's gone; the longest time interrupts stay masked is
measured.
**Read:** CM4: memory model, DMB/DSB/ISB, LDREX/STREX. Yiu: Synchronisation
chapter.
**Check:** why isn't `volatile` enough? How long may interrupts stay masked?

## Stage 3: peripherals and driver design

### Rung 7: Timers, PWM, input capture (`todo`)
**Goal:** a general-purpose timer as a periodic interrupt, PWM on an LED, and
input-capture measurement of an external pulse.
**Done-test:** the measured pulse width agrees with the logic analyzer to within one
timer tick.
**Read:** DS: General-Purpose Timers (and PWM, if you use the PWM module).
**Check:** how would you measure a pulse width?

### Rung 8: ADC (`todo`)
**Goal:** on-chip temperature sensor, sampled from a timer trigger.
**Done-test:** a steady sample rate (verified), temperature printed in °C, and the
conversion formula derived from the datasheet.
**Read:** DS: ADC (sample sequencers, trigger sources, internal temperature sensor).
**Check:** sample rate, aliasing, resolution.

### Rung 9: SPI or I2C with a real sensor, behind a HAL interface (`todo`)
**Goal:** a driver for a real sensor with timeouts and bus-error recovery (stuck SDA,
NACKs); the driver is unit-tested on the host (Unity, mocked registers).
**Done-test:** correct readings; recovery from a NACK and from a pulled-low SDA, shown
on the logic analyzer; `make test` runs the host tests.
**Read:** DS: I2C or SSI. The sensor's datasheet.
**Check:** how do you debug a hanging bus? How do you test firmware without hardware?

### Rung 10: µDMA (`todo`)
**Goal:** UART TX or ADC to memory over µDMA.
**Done-test:** the same transfer as before with CPU load measured before and after DMA.
**Read:** DS: Micro Direct Memory Access (channel control table, transfer modes).
**Check:** DMA vs interrupts vs polling.

## Stage 4: structure

### Rung 11: Event-driven state machine (`todo`)
**Goal:** a small device with modes; ISRs post events to a queue; handlers run to
completion (Samek's active-object style).
**Done-test:** a state diagram on paper that matches the code, and every transition
reachable from a test or a button sequence.
**Read:** Miro Samek, *Modern Embedded Systems Programming* (state machine and
active-object lessons).
**Check:** how do you structure firmware beyond a super-loop?

### Rung 12: Watchdog, assertions, fault logging (`todo`)
**Goal:** a watchdog, `assert` that records file and line, and a fault log that survives
a reset.
**Done-test:** a deliberate hang triggers a watchdog reset, and after the reboot the
reason for the reset and the last fault are printed.
**Read:** DS: Watchdog Timers, System Control (reset cause), Internal Memory (if
logging to flash or EEPROM).
**Check:** what should happen when firmware hangs in the field?

## Stage 5: mini RTOS

### Rung 13: Context switch (`todo`)
**Goal:** task control blocks, PendSV, SysTick tick, round-robin between two threads.
**Done-test:** two tasks toggling two pins at independent rates, plus a GDB walkthrough
of one switch that shows the saved and restored registers.
**Read:** CM4: PendSV, PSP/MSP, CONTROL, exception return (EXC_RETURN). Yiu: the
OS support chapter.
**Check:** walk through a context switch register by register.

### Rung 14: Blocking and priorities (`todo`)
**Goal:** sleep and blocking, then a priority scheduler.
**Done-test:** a high-priority task wakes on time while a low-priority task spins,
with the jitter measured.

### Rung 15: Semaphore, mutex, queue (`todo`)
**Goal:** the three primitives, with a priority inversion reproduced and then fixed by
priority inheritance.
**Done-test:** an inversion trace (logic analyzer) before and after the fix.
**Check:** mutex vs semaphore; races and deadlock on one core with interrupts.

## Stage 6: system

### Rung 16: Bootloader and firmware update (`todo`)
**Goal:** a bootloader at the start of flash, CRC-checked images, A/B slots, update
over UART, jump to the app (VTOR, stack pointer); then the MPU and privileged vs
unprivileged mode.
**Done-test:** an update over UART works; a corrupted image is rejected; pulling power
mid-update leaves the board booting the old image.
**Read:** DS: Internal Memory (flash programming). CM4: VTOR, MPU, CONTROL.
**Check:** how does secure boot extend this? What happens if power dies mid-update?

### Rung 17: CAN (`todo`)
**Goal:** loopback mode first, then two nodes (a second board or a USB-CAN adapter).
**Read:** DS: Controller Area Network (CAN) Module.
**Check:** arbitration, bit stuffing, error frames.

### Rung 18: Low power and CI (`todo`)
**Goal:** sleep and deep-sleep with measured current; GitHub Actions building the
firmware and running the host tests.
**Check:** what dominates an MCU's power budget?

---

## Later (not part of this ladder)
- Port the drivers to an STM32 Nucleo, to learn where the HAL boundary should sit.
- Stanford CS 140E (bare metal on a Raspberry Pi).
- Zephyr: compare its driver model with your own drivers, then send one upstream patch.
