## Overview

CISC8 is a 16-bit variable-length CISC ISA for FPGA soft cores and bare-metal
MCUs. Primary targets are the RP2040, ESP32-S3, and Tang Nano series (9K/4K/1K).
On FPGA, the core is small enough for the Tang Nano 1K (1,152 LUT4).
Bottom-up estimates for a bare peripheral set (UART + timer + GPIO port):

| Config | Core | Divider | Periph | Total | Device fit |
|--------|------|---------|--------|-------|------------|
| T1 | 190-235 | 6-22 | 121 | 317-378 LUT | Nano 1K (28-33%) |
| T2 | 240-295 | 6-22 | 121 | 367-438 LUT | Nano 4K (8-10%) |
| T3 | 330-390 | 10-26 | 121 | 461-537 LUT | Nano 4K (10-12%) |
| T4 (LUT MUL) | 390-460 | 10-26 | 121 | 521-607 LUT | Nano 4K (11-13%) |
| T4 (DSP MUL) | 330-400 | 10-26 | 121 | 461-547 LUT | Nano 4K (10-12%) |

Peripheral breakdown: UART 60 LUT, timer 53 LUT, GPIO port 8 LUT.
Add SysTick (+16), second timer (+34), DMA (+67) as needed.

Core build-up: decoder 80-100 LUT shared across tiers; ALU and feature
increments are additive. T5 (1-cycle MUL pipeline) estimated at ~450-530 LUT.

The instruction set is prefix-free: the first byte fixes the instruction
length, so the decoder never speculates. Instructions are 1-4 bytes; the
1-byte turbo tier covers the common register-to-register and small-immediate
cases. All indirect addressing modes include post-increment and pre-decrement,
so stack and array loops need no separate address-update instructions.

Two independent hardware stacks (S0/S1) are built into the register file.
CALL and RET operate on S0 by default; data stacks use S1. This makes
Forth-style dual-stack code native, not emulated.

The architecture scales from a minimal T0 core (no multiply, no interrupts)
up to T5 (full extension set). A single STRICT parameter gates whether
unsupported encodings trap or execute as NOP, so one binary can run across the
tier range.

Target frequencies on Tang Nano (Gowin 28nm): 27-40 MHz practical after
place-and-route; theoretical ceiling ~150 MHz (BRAM-limited). Tang Nano 20K
(12nm) reaches 50-80 MHz.

The simulator runs on four hardware targets:

| Board | MCU | Role |
|-------|-----|------|
| Luckfox Pico Mini B | Rockchip RV1103, Cortex-A7 @ 1.2 GHz, 64 MB DDR2 | Primary sim host; runs platform_posix unchanged |
| RP2040-Matrix | RP2040, dual Cortex-M0+ @ 133 MHz, 264 KB SRAM | Bare-metal sim proof-of-concept |
| T-Display | ESP32, dual LX6 @ 240 MHz, 320 KB SRAM + PSRAM | Portable sim with onboard ST7789 display and WiFi OTA |
| ESP32-C3 OLED | ESP32-C3, RISC-V @ 160 MHz, 400 KB SRAM | WiFi OTA .ihex loader and debug terminal |

The C simulator costs roughly 40 host instructions per simulated instruction
(1-byte turbo ~20, 2-byte ~40, peripheral tick ~10 idle overhead). Estimated
simulated throughput at --trace=0:

| Board | Host IPC | Sim MIPS | Equiv. FPGA clock |
|-------|----------|----------|-------------------|
| Luckfox (Cortex-A7, OoO) | ~2.0 | ~50-60 | ~250-300 MHz |
| T-Display (LX6 @ 240 MHz) | ~1.0 | ~6 | ~30 MHz |
| ESP32-C3 (RISC-V @ 160 MHz) | ~1.0 | ~4 | ~20 MHz |
| RP2040 (Cortex-M0+ @ 133 MHz) | ~0.9 | ~3 | ~15 MHz |

"Equiv. FPGA clock" is the T1-tier FPGA frequency that would execute at the
same MIPS (10 MIPS per 50 MHz for T1 blended workload).
Luckfox is the only target fast enough for real-time interactive use.

## Co-processors

Three CISC8-adjacent engines share the bus and address space alongside any
main core:

**Tp (Protocol core)** -- cycle-deterministic bit-level I/O. Analogous to the
RP2040 PIO. Up to 8 instances per main core. Each Tp runs a 2-byte-per-word
CISC8 subset (turbo + G opcodes) with an inline wait field for precise
inter-instruction timing. The main core pre-loads a program and timing
constants into Tp via MMIO, then releases it; Tp runs to completion and
signals via CORE_ENABLED. Used for SPI, I2C, 1-Wire, WS2812B, and custom
protocols without interrupting the main core.

**Tf (Forth core)** -- a T3-class CISC8 core with banking removed and F
extension opcodes always active. Designed to run the eForth REPL standalone.
The F extension compresses NEXT/DOCOL/EXIT to 1-2 bytes and 1-3 cycles via a
dedicated RS SRAM port. Tf can be the sole execution engine (no main core
needed); the main core, if present, activates it via CORE_ENABLED and then
steps aside.

**Tj (J1 co-processor)** -- a J1-derived inner interpreter for Forth compound
word dispatch. Activated by the main core; handles colon-word bodies at 1 cycle
per FIW vs 14 cycles on a T5 main core.

| co-processor | LUT | FF | BRAM | purpose |
|---|---|---|---|---|
| Tp (no G) | 125-175 | ~88 | 0 | protocol offload |
| Tp + G ext | 160-215 | ~96 | 0 | protocol + GPIO shift ops |
| Tf | 443-513 | ~132 | 1 | standalone Forth REPL |
| Tj | ~252 | ~96 | 2 | Forth inner interp offload |



### Assembler (asm/)

SDCC/ASXXXX assembler syntax supported:

- `NAME = VALUE` as EQU
- `.module`, `.area`, `.extern` directives
- `.tier T0`-`T5` with extension args; `.tier Tp` errors (co-processor, not yet supported)
- `.ds` / `.blkb` / `.blkw` unified
- All integration tests pass; asm/README.md documents all directives

Turbo INC/DEC (0x08-0x0F) do not update flags, making `dec z / bne` silently
wrong. The assembler will reject these mnemonics and require `add rr, #1` /
`sub rr, #1` instead.

### Simulator (sim/)

Interrupt taxonomy implemented:

- NMI: MPU violation fires fault line (MPU_FAULT_ADDR read-only at 0xFD30-0xFD31)
- Hard IRQ: external interrupt controller
- Soft IRQ: SOFT_IRQ write at 0xFCE2

162 tests pass.

Stats feature complete:

- `--stats` / `--stats=<file>` -- collect run statistics (disabled by default)
- Top-30 instruction frequency, 256x256 UTF-8 execution heatmap, memory access
  patterns, branch statistics per condition code

Current sim options summary:

```
--tier=<0-5>         initial tier (default 1)
--max-steps=<n>      halt after n instructions retired
--max-cycles=<n>     halt after n cycles
--trace=<0-6>        trace verbosity; 6 = full fetch+mem
--trace=<l>:<s>:<e>  windowed trace by step count or PC (0x prefix = PC)
--uart-in=<str>      pre-load UART0 RX FIFO
--uart-in-file=<f>   pre-load UART0 RX from file
--uart-eof           append 0x04 (EOF) after uart-in (opt-in)
--code-rw            disable write-protection of code range
--qext               lax mode: unsupported encodings are NOP
--stats[=<file>]     collect and write run statistics
--vcd[=<file>]       emit VCD trace
```

### Forth Kernel

`isa/analysis/forth_kernel_t1.s` -- Direct Threaded Code, T1, fully assembled.

Register convention:

| reg | role |
|-----|------|
| Z   | IP (instruction pointer) |
| T   | W (code address / scratch in NEXT) |
| X   | TOS (cached) |
| Y   | scratch |
| S0  | RSP (return stack, grows down from 0xEFFE) |
| S1  | DSP (data stack, grows down from 0xE7FE) |

E2E status (as of 2026-05-05):

| expression        | result              | status |
|-------------------|---------------------|--------|
| `2 3 + .`         | `00005`             | PASS   |
| `.s` (empty)      | `( )`               | PASS   |
| `10 3 -`          | `00007`             | PASS   |
| `6 7 *`           | `00042`             | PASS   |
| `10 fib`          | `00055`             | PASS   |
| `5 dup .s`        | `( 00005 00005 )`   | PASS   |
| `1 2 swap .s`     | `( 00001 00002 )`   | PASS   |
| `1 2 3 drop .s`   | `( 00002 00001 )`   | PASS   |
| `1 2 over .s`     | `( 00001 00002 00001 )` | PASS |

Known gaps: `/` (no divmod), colon definitions (compile path failing),
`>r` / `r>` (hang).

## Quick Start

Assemble and run:

```sh
cd asm && ./asm kernel.s -o out/kernel.ihex
cd ../sim && ./cisc8_sim out/kernel.ihex --tier=1 --max-steps=500000
```

Loader auto-detects format: `:` = ihex, `X` = .rel, other = flat binary.

Run with stats and UART input:

```sh
./cisc8_sim ../asm/out/forth_kernel_t1.ihex \
    --tier=1 --max-steps=500000 \
    --uart-in=$'2 3 + .\n' --uart-eof \
    --stats
```


## LUT Budgets (reference)

Single-core, bare peripherals (UART + timer + GPIO). Divider included.

| Config | Core | +Divider | +Periph | Total LUT | Device | Headroom |
|--------|------|----------|---------|-----------|--------|----------|
| T1 | 190-235 | 6-22 | 121 | 317-378 | Nano 1K (1,152) | 67-73% |
| T2 | 240-295 | 6-22 | 121 | 367-438 | Nano 1K (1,152) | 62-68% |
| T3 | 330-390 | 10-26 | 121 | 461-537 | Nano 4K (4,608) | 88-90% |
| T4 (DSP MUL) | 330-400 | 10-26 | 121 | 461-547 | Nano 4K (4,608) | 88-90% |
| T4 (LUT MUL) | 390-460 | 10-26 | 121 | 521-607 | Nano 4K (4,608) | 87-89% |
| T5 | 450-530 | 10-26 | 121 | 581-677 | Nano 9K (8,640) | 92-93% |

Tang Nano 9K has hard DSP blocks; use DSP MUL figure for T4 on that device.
G extension adds +38 LUT per core.

### Multicore (M extension)

Multicore SMP is supported from T0 upward. The M extension adds a round-robin
bus arbiter and per-pair mailbox registers. Cores share one bus; each sees a
sequential-consistency memory model. Core 0 brings up additional cores via the
CORE_ENABLED MMIO register.

Bottom-up for 2-core configs, shared peripherals (UART + timer + GPIO):

| Config | 2x Core | +Arbiter | +Divider | +Periph | Total LUT | Device |
|--------|---------|----------|----------|---------|-----------|--------|
| 2x T1 | 380-470 | 60-100 | 6-22 | 121 | 567-713 | Nano 4K (12-15%) |
| 2x T2 | 480-590 | 60-100 | 6-22 | 121 | 667-833 | Nano 4K (14-18%) |
| 4x T1 | 760-940 | 140-220 | 6-22 | 121 | 1027-1303 | Nano 9K (12-15%) |
| 4x T2 | 960-1180 | 140-220 | 6-22 | 121 | 1227-1543 | Nano 9K (14-18%) |

Arbiter cost: ~30-40 LUT base + ~15 LUT per additional core. Mailbox adds
~5 LUT / ~17 FF per core pair. Each core gets its own divider tap (shared
counter, no extra LUT). Peripherals counted once (shared bus).
