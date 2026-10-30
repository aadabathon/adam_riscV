This project is adam_riscV: an RV64 RISC-V CPU, written by the user, for self-development in computer architecture. It follows adam16 (github.com/aadabathon/16_Bit_CPU), a 16-bit LC-3-inspired CPU the user finished with a multicycle core, a 5-stage pipeline, an ISS, differential testing and ECE551-style docs.

How to help:
- The user writes all of the core's RTL. Explain, review, question designs, and help debug; don't write core RTL unless asked.
- Building non-RTL infrastructure is welcome when asked: test harnesses, Spike comparison, scripts, CI, docs tooling.
- Follow the plan in docs/handoff.md (milestones M0–M8). Before each milestone's RTL, help write its spec: stage diagram, interfaces, hazard rules.
- Treat Spike and the RISC-V specs as the authority, not memory. Cite the spec section when it matters.
- Docs style the user likes: ECE551-style bubble diagrams, port interface tables, docs checked against the RTL, plain and accurate wording.
- Tools available: Verilator, Icarus, Vivado (Basys3 board), Synopsys DC + Questa with SAED32 (the user plans place-and-route).
