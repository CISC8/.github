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
