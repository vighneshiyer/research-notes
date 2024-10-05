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

- Ansh and Pramath will do the first RISC-V spike Rust prototype
  - Initially interpret just rv64ui
  - Support memory ops
  - Hand write assembly tests (or just a shim on top of riscv-tests default env - see 151 tests)
  - Get spike diff testing working very first
- Safin: fesvr integration and rewrite side
- Junha: Vienna
