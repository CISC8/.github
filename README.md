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

Target frequencies: 80-125 MHz on iCE40, 100-150 MHz on ECP5.

## Current State

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

## Repo Structure

Five git repos, one per component:

| path     | repo   | content                        |
|----------|--------|--------------------------------|
| `isa/`   | isa    | spec, ADRs, analysis           |
| `sim/`   | sim    | C simulator                    |
| `asm/`   | asm    | assembler                      |
| `sdcc/`  | sdcc   | SDCC backend (archived)        |
| `tools/` | tools  | shexec and build helpers       |
| `www/`   | www    | project website                |

ADRs live in `isa/adr/`. Next ADR number: 071.
Index and template: `isa/adr/README.md`.

## LUT Budgets (reference)

| configuration                      | LUT   | FF   | target            |
|------------------------------------|-------|------|-------------------|
| T0M + 2x TzGM                      | 315   | 170  | Tang Nano 1K      |
| T5GM + 3x T3GM + 4x TzGM          | 1962  | 156  | Tang Nano 9K (23%) |
| G extension overhead               | +21   | --   | per core          |
