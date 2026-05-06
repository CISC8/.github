## Overview

CISC8 is a 16-bit variable-length CISC ISA targeting small FPGA soft cores
(200-350 LUT on iCE40) and embedded MCUs. Optimised for code density and
simple hardware decode.

Design targets:

- code density competitive with STM8, significantly better than RV32E
- single-cycle execution for all instructions
- prefix-free variable-length encoding (first byte determines length)
- post-increment and pre-decrement addressing built into all indirect ops
- SP-relative addressing for efficient C local variable access
- sub-register byte-half access via W prefix and F register MASK field
- clean indirect jump and call for Forth and C function pointers
- 80-125 MHz on iCE40, 100-150 MHz on ECP5

## Current State (v16, 2026-05-06)

### Spec

v15 is frozen. v16 is the active spec at `isa/v16/`.

Key v16 changes from v14/v15:

- ADR-023: FNEXT DTC (PC=T, 1 cycle) in cisc8_ext_f_v16.md
- ADR-028: Turbo opcode map 0x18-0x2F/0x37 finalised
- ADR-017: ST imm8 retired
- ADR-061: CALL_SP_SEL at 0xFD2A; CALL/RET use S0 by default
- ADR-062: .tier 0 with ssp1 is a hard assembler error
- S2/S3 dropped from T5 (encoding unchanged, opcodes trap; saves ~64 LUT)
- Extension matrix, opcode map (43 free slots identified), changelog all updated

Pending prose ADRs (Proposed, not yet applied to spec narrative):
ADR-044 (tier design principles), ADR-046 (stack arch rationale),
ADR-047 (T0 scope clarification).

Deferred to v17+: ADR-018, ADR-024, ADR-025, ADR-037/038/039/040.

### Assembler (asm/)

ADR-063 complete. SDCC/ASXXXX assembler syntax fully supported:

- `NAME = VALUE` as EQU; `.equ` removed
- `.module`, `.area`, `.extern` directives
- `.tier T0`-`T5` with extension args; `.tier Tp` errors (co-processor, not yet supported)
- `.ds` / `.blkb` / `.blkw` unified under DIR_DS
- All integration tests migrated; asm/README.md (307 lines) documents all directives

ADR-070 (inc/dec rejection) is Proposed: turbo INC/DEC (0x08-0x0F) do not
update flags, making `dec z / bne` silently wrong. Plan is to reject the
mnemonics in the assembler and require `add rr, #1` / `sub rr, #1`.

### Simulator (sim/)

ADR-067 complete. Interrupt taxonomy implemented:

- NMI: MPU violation fires fault line (MPU_FAULT_ADDR read-only at 0xFD30-0xFD31)
- Hard IRQ: external interrupt controller
- Soft IRQ: SOFT_IRQ write at 0xFCE2 fires software interrupt
- PWR_PERIPH, CORE_ENABLED stub MMIO registers added
- All 162 tests pass

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
--qext               lax mode: unsupported encodings are NOP (ADR-037)
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
