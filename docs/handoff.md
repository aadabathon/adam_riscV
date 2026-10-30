# adam_riscV: handoff

The plan and background for an RV64 RISC-V CPU project, written at the end of the adam16 project. The user writes the RTL; Claude explains, reviews, verifies and builds infrastructure when asked.

---

## 1. Background: adam16

Repository: https://github.com/aadabathon/16_Bit_CPU (final state in PR #2).

- **ISA:** 16-bit, LC-3-inspired, frozen. Word-addressed 64K memory, R0–R7, N/Z/P flags, signed CMP, TRAP as a direct jump, `TRAP x025` as the HALT convention.
- **Two cores:**
  - **Multicycle** (`adam16_core`): 14-state FSM plus datapath, 5–8 cycles per instruction.
  - **5-stage pipeline** (`adam16_pipe_core`): dual-port block RAM, full forwarding, no stalls (the data address goes out in EX, so load data forwards from MEM), one bubble per taken branch. `sum_array`: 191 → 43 cycles; CPI 5.55 → 1.16 on random programs.
- **Verification:**
  - Python ISS as the golden model; ISS and both cores compared instruction by instruction.
  - Retire-trace format: `pc ir reg_we reg_addr reg_data mem_we mem_addr mem_data nzp`.
  - Directed `.asm` tests with `;!` expectations, plus a random-program generator.
  - Exhaustive conformance checks: the FSM against its spec, the decoder against its spec.
  - Mutation testing: bugs planted one at a time to prove the regression catches them.
- **Docs:**
  - ECE551-style bubble diagrams generated from a Python spec, checked against the RTL.
  - Interface sheets with symbols and connection tables generated from the RTL.
  - Port tables checked by a script; link checker.
  - Everything runs under `make check` in CI.
- **Board / ASIC:**
  - Basys3: 100 MHz clock, the step button is a clock enable; `fpga/build.tcl` is a project-free Vivado build.
  - Design Compiler and Questa scripts driven by environment variables.
  - The Vivado, DC and board runs were never done in CI; the user runs them.

### Lessons that carry over

- **Memory latency.** adam16 assumed fixed-latency memory. For RISC-V, use a valid/ready memory interface from day one, or caches will force a rewrite.
- **Wrong-path stores are invisible to a retire-trace comparison.** Read the data back in the test, or compare final memory.
- **Mutation-test the regression.** Before trusting a test suite, break the RTL on purpose and check that it fails.
- **Toolchain.** Yosys 0.33 (Ubuntu 24.04) can't parse SystemVerilog packages with struct-returning functions. Use yosys-slang or sv2v if riscv-formal is wanted.
- **Lint.** Keep Verilator `-Wall` clean from the first commit.
- **Docs tooling reuse.** The adam16 docs tooling (`docs/tools/`: RTL header parser, symbol generator, interconnect tables, port-table checker, link checker) can be copied. It expects one port per line in module headers.

---

## 2. Decisions

| | Choice | Why |
|---|---|---|
| ISA | **RV64**, grown I → M → Zicsr/M-mode → A → S/U + Sv39. No F/D/C | Each step teaches a distinct part of a real computer; Linux builds without F/D/C |
| Microarchitecture | In-order 5-stage pipeline | Known from adam16; new learning goes into everything around it |
| Memory interface | Request/response with valid/ready from day one, even backed by block RAM | Caches and DDR break fixed-latency assumptions |
| Retire interface | **RVFI** from day one | Feeds Spike lockstep, riscv-formal and debug tools |
| Simulator | Verilator + C++ testbench | Speed for CoreMark and, later, booting Linux in simulation |
| Boards | Basys3 for M1–M6; a DDR board (Arty A7-100T or Nexys A7-100T) for M7 | Basys3 has ~225 KB block RAM, too little for Linux |
| Purpose | Self-development; no compatibility pressure | Take every shortcut the spec allows |

## 3. Ground rules

1. **Spec before RTL** for every milestone: stage diagram, interfaces, hazard rules.
2. **Verify against Spike**, not against expectations: every retired instruction must match its commit log.
3. **Use the standard suites:** `riscv-tests` and `riscv-arch-test`. Don't hand-write a test per instruction.
4. **Measure** CPI and CoreMark/MHz from M3 on, and record them after every change.
5. **Keep a design journal:** what broke, why, how it was found.

---

## 4. Milestones

**M0. Harness, no core yet.** Size S.
- Build:
  - `riscv64-unknown-elf-gcc`, Spike, and `riscv-tests`.
  - A Verilator testbench skeleton.
  - A retire-trace comparison script.
  - CI.
- Done when: a stub core runs one `rv64ui-p-*` test and the harness reports pass/fail through `tohost`.

**M1. Single-cycle RV64I (simulation only).** Size S.
- Build:
  - Decoder and immediate generation.
  - ALU, including the `*W` ops.
  - Branch unit and load/store alignment.
  - Combinational memory in the testbench.
- Learn: the encodings, without pipeline confusion.
- Done when: every `rv64ui-p-*` test passes. Keep the decoder and ALU for M2.

**M2. Five-stage pipeline.** Size M.
- Build:
  - IF ID EX MEM WB with a hazard unit (forwarding, load-use stall).
  - Branch resolution and flushes.
  - Stalls driven by memory valid/ready.
- Learn: stall and flush as one central mechanism.
- Done when:
  - `rv64ui` and `riscv-arch-test` RV64I pass.
  - Thousands of random programs match Spike.
  - Optionally, riscv-formal passes.

**M3. A small system on the Basys3.** Size M.
- Build:
  - Boot ROM, block RAM and a UART.
  - A memory map, linker script and C runtime startup.
- Done when: a C program prints over USB-UART, and the Dhrystone and CoreMark baselines are recorded.

**M4. M extension.** Size S.
- Build:
  - A DSP-based multiplier.
  - An iterative divider (~64 cycles), giving variable execution latency.
- Learn: structural hazards and multicycle functional units.
- Done when: `rv64um` passes and the CoreMark change is measured.

**M5. Machine mode: CSRs, traps, interrupts.** Size L.
- Build:
  - `mstatus`, `mtvec`, `mepc`, `mcause`, `mtval`, `mie`/`mip`.
  - ECALL, EBREAK and MRET.
  - Illegal-instruction and misaligned traps.
  - A CLINT (`mtime`/`mtimecmp`).
  - `mcycle`/`minstret`.
- Learn: precise exceptions. Take traps at one commit point and flush everything younger. Serialize CSR instructions at first.
- Done when: `rv64mi` passes and FreeRTOS runs two tasks switched by timer interrupts.

**M6. Performance.** Size L. Measure each step with CoreMark.
- Build:
  1. Branch prediction: BTB + 2-bit counters, then gshare, then a return-address stack.
  2. Caches: I-cache, then a write-back D-cache; direct-mapped first, then 2-way.
  3. A real bus (AXI4-Lite or custom), plus hardware performance counters.
- Learn: Hennessy & Patterson chapters 2–3, by building it.
- Done when: there is a table of CoreMark/MHz, hit rates and misprediction rates after each feature.

**M7. Operating-system capable: boot Linux.** Size L+. The capstone.
- Build:
  - A extension: LR/SC and AMOs (single core: locked read-modify-write in the D-cache).
  - S/U modes with `medeleg`/`mideleg`.
  - Sv39: I/D TLBs, a hardware page-table walker, `sfence.vma`.
  - `fence.i` (I-cache flush).
  - PLIC and a 16550-compatible UART.
  - DDR via MIG or LiteDRAM on the new board.
- Shortcuts:
  - OpenSBI emulates misaligned accesses and `rdtime` in software.
  - Build Linux + BusyBox with Buildroot for `rv64ima`.
- Done when:
  - `rv64si`, `rv64ua` and `rv64ui-v-*` pass.
  - OpenSBI prints its banner.
  - Linux reaches a shell, in Verilator first, then on the board.

**M8. Optional directions afterwards.**
- Out-of-order: register renaming, a reorder buffer, Tomasulo-style issue.
- A dual-issue superscalar.
- Two cores with cache coherence.
- F/D floating point.
- ASIC place-and-route of the M3 core with SAED32.

**Scale:** M0–M3 is about adam16's size. M5 and M7 are each comparable to everything before them. Linux is months of evenings; every milestone is a finished stopping point.

---

## 5. Verification, every milestone

| Tool | Catches |
|---|---|
| `riscv-tests`, `riscv-arch-test` | Wrong instruction semantics |
| Spike lockstep (`--log-commits` vs the RVFI trace) | First divergent instruction |
| Random instruction programs | Hazard bugs nobody thought to test |
| riscv-formal (Yosys; yosys-slang or sv2v for SystemVerilog) | Corner cases no test reaches |
| SVA assertions (e.g. no instruction retires twice) | Broken invariants, at the source |

## 6. Pitfalls

- **Fixed-latency memory assumptions.** Removing them later is the costliest refactor in the project.
- **CSR and trap logic spread across stages.** Pick one commit point at M5 and keep it.
- **Unneeded generality.** RV64 only; no configurable-XLEN core.
- **Reading reference designs too early.** Read Ibex, CVA6 and VexRiscv only after building your own version of a feature.

## 7. Reading

- *Digital Design and Computer Architecture, RISC-V Edition* (Harris & Harris): M1–M2.
- *Computer Organization and Design, RISC-V Edition* (Patterson & Hennessy): M2–M6.
- *Computer Architecture: A Quantitative Approach* (Hennessy & Patterson): M6, M8.
- RISC-V Unprivileged and Privileged specifications: everything. The Privileged spec is essential for M5 and M7.
- *Modern Processor Design* (Shen & Lipasti): if taking the out-of-order path.

## 8. Open for the first session

- Create the repository and decide its name.
- Decide whether Claude sets up M0 (toolchain, Spike comparison, Verilator harness, CI) so the user can start straight on RTL.
- The DDR board for M7 can wait until M6.
