
Master simulation, ganged simulation for dv, slave simulation as trace ingester, single inst replay stepper, symbolic execution, many modes we want to support

Switch to dbt simulator mode, if possoble
NEW
10:31
Exact soc modeling, checkpoint and replay, switch simulation modes during runtime
10:31
Ability to transpile into generic isa ir during execution for generic pass writing
10:31
Maybe that should be a part of tracekit instead
10:33
Create a very high perf riscv disassembler that disassembles into rust native structures either based on inst type (r, I, etc) or semantic inst type (arith, mem, etc)
10:33
Leverage simd

- fesvr + io models + everything on the edge needs to work in RTL sim + FPGA emulation / firesim + functional simulation
  - need to make top-level ports explicit, no internal DPIs
- need to do review of Vienna - someone should look into that
  - https://arxiv.org/pdf/2402.09087
  - [Cycle-Accurate Simulator Generator for the VADL Processor Description Language](https://repositum.tuwien.at/bitstream/20.500.12708/17053/1/Schuetzenhoefer%20Hermann%20-%202020%20-%20Cycle-Accurate%20simulator%20generator%20for%20the%20VADL...pdf)
  - [Optimized Processor Simulation with VADL](https://repositum.tuwien.at/bitstream/20.500.12708/157928/1/Mihaylov%20Hristo%20-%202023%20-%20Optimised%20Processor%20Simulation%20with%20VADL.pdf)

- Ansh and Pramath will do the first RISC-V spike Rust prototype
  - Initially interpret just rv64ui
  - Support memory ops
  - Hand write assembly tests (or just a shim on top of riscv-tests default env - see 151 tests)
  - Get spike diff testing working very first
- Safin: fesvr integration and rewrite side
- Junha: Vienna


- specification first interpreter/jit generator for ISA simulation
- look at NEMU and Vienna
  - https://stackoverflow.com/questions/75028678/is-it-impossible-to-write-thread-code-in-rust
  - https://users.rust-lang.org/t/how-can-i-approach-the-performance-of-c-interpreter-that-uses-computed-gotos/6261
  - https://stackoverflow.com/questions/58774170/how-to-speed-up-dynamic-dispatch-by-20-using-computed-gotos-in-standard-c
  - https://www.complang.tuwien.ac.at/forth/threaded-code.html

Everyone should be familiar with compiling spike (https://github.com/riscv-software-src/riscv-isa-sim) from source, running the riscv-tests (https://github.com/riscv-software-src/riscv-tests). It would be helpful to understand how a RISC-V target is tethered to the host in spike using fesvr (https://chipyard.readthedocs.io/en/latest/Advanced-Concepts/Chip-Communication.html#using-the-tethered-serial-interface-tsi-or-the-debug-module-interface-dmi) - there is a short writeup, but unfortunately this is not well documented.

https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9923860
Look at this paper, in particular the section "NEMU: Fast Interpreter for Performance Evaluation"

Get familiar with threaded interpreters
https://www.complang.tuwien.ac.at/forth/threaded-code.html
https://stackoverflow.com/questions/58774170/how-to-speed-up-dynamic-dispatch-by-20-using-computed-gotos-in-standard-c
https://stackoverflow.com/questions/3848343/decode-and-dispatch-interpretation-vs-threaded-interpretation

dromajo vs spike
use riscv-tests to difftest with spike unmodified
make sure we consider cosim with ganged isa sim as we're developing this
- leverage the same fesvr + IO models across bringup/chip, RTL sim, functional sim, emulation platform, FPGA prototyping, Firesim (current state of fesvr today... but not sufficient since we need exact state snapshotting, ideally no C++)
  - same thing with IO models
- task distribution on top
- we need a ultra identical setup in functional sim that matches the SoC exactly, RTL that's generated should be driving the parameterization of the functional sim (not the other way around), pass a dts and bootrom into the functional sim from RTL generator as first-class supported primitive

- Vienna ADL
- CodAL
