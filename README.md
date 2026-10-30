# adam_riscV

An RV64 RISC-V CPU in SystemVerilog, built for self-development in computer architecture. It follows [adam16](https://github.com/aadabathon/16_Bit_CPU), a 16-bit LC-3-inspired CPU with a multicycle core and a 5-stage pipeline.

The plan is in [docs/handoff.md](docs/handoff.md). It's an in-order 5-stage pipeline that grows from RV64I through M, Zicsr/M-mode and A to S/U modes with Sv39. Milestones M0 to M8 end in booting Linux.

## Status

M0 (toolchain and test harness) has not started.

## Layout

| Directory | Contents |
|---|---|
| `rtl/` | Core RTL |
| `tb/` | Verilator testbench and harness |
| `sw/` | Test programs and software |
| `docs/` | Plan, specs and design journal |
