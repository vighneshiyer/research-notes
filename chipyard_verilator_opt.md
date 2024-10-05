# Chipyard Verilator Optimization

## Investigating -O3 vs -Oz

In the [Arcilator (Fast and cycle-accurate hardware simulation in CIRCT) talk](https://www.youtube.com/watch?v=iwJBlRUz6Vw) given at the 2023 LLVM Dev Meeting, the speaker received a question from a Verilator developer.
The Verilator dev claimed that you should always compile Verilator binaries with `-Oz` (aggressively optimize for size) rather than `-O2`/`-O3` (optimize for speed) since Verilator simulations (any RTL simulation really) is frontend bound and would benefit from lower icache pressure.

So let's investigate this.
The setup is the latest Chipyard, which by default uses `-O3` for `gcc`.
The benchmark is `dhrystone`.

The configurations I tested were:

- With default -O3

```bash
hyperfine "/scratch/vighneshiyer/chipyard-verilator/sims/verilator/simulator-chipyard.harness-FastRTLSimRocketConfig \
                               +permissive \
                               +dramsim +dramsim_ini_dir=/scratch/vighneshiyer/chipyard-verilator/generators/testchipip/src/main/resources/dramsim2_ini +max-cycles=10000000   +loadmem=/scratch/vighneshiyer/chipyard-verilator/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/benchmarks/dhrystone.riscv  \
                               +permissive-off \
                               /scratch/vighneshiyer/chipyard-verilator/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/benchmarks/dhrystone.riscv"
Benchmark 1: /scratch/vighneshiyer/chipyard-verilator/sims/verilator/simulator-chipyard.harness-FastRTLSimRocketConfig         +permissive         +dramsim +dramsim_ini_dir=/scratch/vighneshiyer/chipyard-verilator/generators/testchipip/src/main/resources/dramsim2_ini +max-cycles=10000000   +loadmem=/scratch/vighneshiyer/chipyard-verilator/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/benchmarks/dhrystone.riscv          +permissive-off         /scratch/vighneshiyer/chipyard-verilator/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/benchmarks/dhrystone.riscv
  Time (mean ± σ):      6.777 s ±  0.323 s    [User: 6.752 s, System: 0.028 s]
  Range (min … max):    6.357 s …  7.327 s    10 runs
```

- With -Os

```bash
hyperfine "/scratch/vighneshiyer/chipyard-verilator-opt/sims/verilator/simulator-chipyard.harness-FastRTLSimRocketConfig \
                               +permissive \
                               +dramsim +dramsim_ini_dir=/scratch/vighneshiyer/chipyard-verilator-opt/generators/testchipip/src/main/resources/dramsim2_ini +max-cycles=10000000   +loadmem=/scratch/vighneshiyer/chipyard-verilator-opt/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/benchmarks/dhrystone.riscv  \
                               +permissive-off \
                               /scratch/vighneshiyer/chipyard-verilator-opt/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/benchmarks/dhrystone.riscv"
Benchmark 1: /scratch/vighneshiyer/chipyard-verilator-opt/sims/verilator/simulator-chipyard.harness-FastRTLSimRocketConfig         +permissive         +dramsim +dramsim_ini_dir=/scratch/vighneshiyer/chipyard-verilator-opt/generators/testchipip/src/main/resources/dramsim2_ini +max-cycles=10000000   +loadmem=/scratch/vighneshiyer/chipyard-verilator-opt/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/benchmarks/dhrystone.riscv          +permissive-off         /scratch/vighneshiyer/chipyard-verilator-opt/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/benchmarks/dhrystone.riscv
  Time (mean ± σ):      7.109 s ±  0.261 s    [User: 7.082 s, System: 0.030 s]
  Range (min … max):    6.675 s …  7.441 s    10 runs
```

- So on first glance, with this small design (FastRTLSimRocketConfig), using -Os degrades performance by a small bit
  - It's possible that for a larger design, like RocketConfig with the TLMonitors, -Os will do better or perhaps even -Oz
