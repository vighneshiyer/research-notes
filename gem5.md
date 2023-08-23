# gem5 Hacking, RTL <-> gem5 Correlation, uArch State Transfer

## 8/22/2023

- [ ] Re-verify that we can compile embench baremetal rv64 and run the benchmarks on gem5 (user-space mode) and spike (with pk)

- Building embench for rv64

```patch
diff --git a/config/riscv32/boards/rv32wallyverilog/board.cfg b/config/riscv32/boards/rv32wallyverilog/board.cfg
index 2103087..d93e364 100644
--- a/config/riscv32/boards/rv32wallyverilog/board.cfg
+++ b/config/riscv32/boards/rv32wallyverilog/board.cfg
@@ -18,8 +18,8 @@ cc = 'riscv64-unknown-elf-gcc'
 # ldflags = (['-Wl,-gc-sections', '-nostdlib', '-march=rv32imac', '-mabi=ilp32', '-T../../../config/riscv32/boards/rv32wallyverilog/link.ld'])
 # cflags = (['-c', '-Os', '-ffunction-sections', '-nostartfiles', '-march=rv32imac', '-mabi=ilp32'])
 # ldflags = (['-Wl,-gc-sections', '-nostartfiles', '-march=rv32imac', '-mabi=ilp32', '-T../../../config/riscv32/boards/rv32wallyverilog/link.ld'])
-cflags = (['-c', '-fdata-sections', '-ffunction-sections', '-march=rv32imac', '-mabi=ilp32'])
-ldflags = (['-Wl,-gc-sections', '-march=rv32imac', '-mabi=ilp32', '-T../../../config/riscv32/boards/rv32wallyverilog/link.ld'])
+cflags = (['-c', '-fdata-sections', '-ffunction-sections', '-march=rv64gc', '-mabi=lp64'])
+ldflags = (['-Wl,-gc-sections', '-march=rv64gc', '-mabi=lp64', '-T../../../config/riscv32/boards/rv32wallyverilog/link.ld'])
 # - cc_define_pattern ('-D{0}')
 # - cc_incdir_pattern ('-I{0}')
 # - cc_input_pattern ('{0}')
diff --git a/config/riscv32/boards/rv32wallyverilog/link.ld b/config/riscv32/boards/rv32wallyverilog/link.ld
index 0c419ba..cf3cdd5 100644
--- a/config/riscv32/boards/rv32wallyverilog/link.ld
+++ b/config/riscv32/boards/rv32wallyverilog/link.ld
@@ -5,8 +5,8 @@
    notice and this notice are preserved.
    Contributor: Daniel Torres <dtorres@hmc.edu>
     */
-OUTPUT_FORMAT("elf32-littleriscv", "elf32-littleriscv",
-	      "elf32-littleriscv")
+OUTPUT_FORMAT("elf64-littleriscv", "elf64-littleriscv",
+	      "elf64-littleriscv")
 OUTPUT_ARCH(riscv)

 ENTRY(_start)
@@ -263,4 +263,4 @@ SECTIONS
 /* MEMORY
 {
   RAM (rwx) : ORIGIN = 0x80000000, LENGTH = 128M
-} */
\ No newline at end of file
+} */
diff --git a/config/riscv32/chips/generic/chip.cfg b/config/riscv32/chips/generic/chip.cfg
index 6632c3e..651655b 100644
--- a/config/riscv32/chips/generic/chip.cfg
+++ b/config/riscv32/chips/generic/chip.cfg
@@ -67,4 +67,4 @@

 # - we garbage collect unused sections on linking

-cc = 'riscv32-unknown-elf-gcc'
+cc = 'riscv64-unknown-elf-gcc'
```

Invocation:

```shell
./build_all.py --arch riscv32 --chip generic --board ri5cyverilator --cc riscv64-unknown-elf-gcc --cflags="-c -O2 -ffunction-sections" --ldflags="-Wl,-gc-sections" --user-libs="-lm" -v
```

- [ ] Re-collect stats from gem5 (IPC, cache miss rates) and make sure they look reasonable
- [ ] Attempt to get spike checkpointing working for the embench binaries (execute them partially in spike and resume execution in gem5)
    - This may not be easy since the spike checkpoint is more than just an .elf, but also contains some extra state that's loaded via DMI, that functionality may not be available in gem5 so an alternative might be necessary
    - https://chipyard.readthedocs.io/en/stable/Advanced-Concepts/Architectural-Checkpoints.html
- Later tasks
    - Correlate gem5 core and uncore parameters with Chipyard parameters (cache hierarchy, sizing, and associativity ; branch predictor state + uarch)
    - Figure out a way to dump uarch state from gem5 and reload it into rtl sim (ideally it should look like the spike checkpoint, but that won't work for uarch state)
    - Figure out a Chisel annotation mechanism to annotate uarch/arch state registers such that we can pick them out in RTL and maybe load them using force (VCS) or verilator_public assignment (Verilator). We just need a way for circt to keep these annotations tied to a signal so that we can see what they map to in lofirrtl.

## 5/2/2023

- embench on rv64, current status - able to get it to compile for rv64 using baremetal and run on gem5 and that works!
    - we also have perf statistics from gem5
- Next: embench rv64 on Rocket, which implies we first need embench rv64 on spike
    - Currently, embench on spike: works with `pk`
    - md5sum (on gem5 perf stats): 2.3M insts, 0.86 CPI (???, odd, Rocket can only commit 1 inst per cycle)
- Next: we need a way to collect perf statistics out of rocket (out-of-band) in RTL simulation
    - Ideally we want this to work on Firesim too
    - Chisel printf (https://www.chisel-lang.org/chisel3/docs/explanations/printing.html)
    - Chipyard (https://chipyard.readthedocs.io/en/stable/Chipyard-Basics/Initial-Repo-Setup.html, https://chipyard.readthedocs.io/en/stable/Simulation/Software-RTL-Simulation.html)
    - Nayiri I think has a perf statistic dumper already somewhere? Using for arch counter based power modeling
        - Just ask her for the branch and diff
    - Build an RTL simulation and validate that we can dump accurate statistics by correlating it with manually observed values from the waveform
    - Next: make sure collecting stats of programs running on pk works just fine, and tune the N-cycles that a statistic is aggregated over
    - Next: make sure the ISA tests correlate first between RTL simulation and gem5 (we need to make sure the arch params that gem5 is configured with are accurate and representative)
    - Next: move on to the embench stuff
- Later: build a generic perf counter framework that can collect OOB stats and correlate with a commit log
- TODO: get Dhruv access to Millennium machines with LDAP creds

## 3/16/2023

- Refer to the gem5 paper below
- Also look at a riscv gem5 evaluation: https://carrv.github.io/2021/papers/CARRV2021_paper_63_Chatzopoulos.pdf
- Also take a look at the gem5 model configuration for a real SiFive board: https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/src/python/gem5/prebuilt/riscvmatched/riscvmatched_core.py#85
- Strober paper: https://dl.acm.org/doi/pdf/10.1145/3007787.3001151
- SimPoint stuff: https://cseweb.ucsd.edu/~calder/simpoint/
- Try to play with gem5 riscv - build a simple in-order core model (that looks like your 151 CPU) and run a riscv binary through it

## 2/7/2023

- Hansung (maybe GPU RTL), Abe (Chipyard various things)
- Baseline understanding of tools
    - Chisel (https://github.com/freechipsproject/chisel-bootcamp (Ch. 1-3), https://github.com/ucb-bar/chisel-tutorial)
    - Chipyard (building RTL, building an RTL simulator - Verilator / VCS, ran a RISC-V binary through it, insn retirement log)
        - https://github.com/ucb-bar/chipyard
        - https://chipyard.readthedocs.io/en/stable/ (look through Ch 1 and 2) - be able to build a Verilator RTL simulator of the default Chipyard SoC config and run some riscv ISA tests through the RTL simulator
    - ISS (inside Chipyard, there is a ISA simulator called spike - spike can also run RISC-V binaries, it will also give you an insn retirement log)
        - https://github.com/riscv-software-src/riscv-isa-sim

- TODOs:
    - Build spike from source (in Chipyard)
    - Run spike on RISC-V binaries (chipyard/toolchains/riscv-tools/riscv-tests)
    - Check the cmdline arguments of spike and get a insn retirement log out
    - Check this against the log from RTL simulation

- Read through this guide, and try compiling gem5 and running x86 binaries through it, and get out a performance trace: https://www.gem5.org/documentation/learning_gem5/introduction/
- Investigate how to get the RISC-V port of gem5 working, and run RISC-V binaries through it with a baseline CPU uArch model (we can just model a simple in-order 5 stage pipeline with the typical bypass paths you're used to)

- Relevant papers:
    - Architectural Simulators Considered Harmful (https://ieeexplore.ieee.org/abstract/document/7155440) - a critique of arch sim, and their accuracy, and why it is better to evaluate RTL, and if you were to build an arch sim - why you should have knowledge of RTL design to begin with
    - gem5 papers:
        - The gem5 simulator: https://dl.acm.org/doi/abs/10.1145/2024716.2024718
        - Micro-architectural simulation of in-order and out-of-order ARM microprocessors with gem5: https://ieeexplore.ieee.org/abstract/document/6893220
    - risc-v gem5 papers:
        - RISC5: Implementing the RISC-V ISA in gem5: https://carrv.github.io/2017/papers/roelke-risc5-carrv2017.pdf
        - Simulating Multi-Core RISC-V Systems in gem5: https://www.csl.cornell.edu/~cbatten/pdfs/ta-gem5-riscv-slides-carrv2018.pdf
        - Towards Accurate Performance Modeling of RISC-V Designs(https://carrv.github.io/2021/slides/CARRV2021_slides_63_Chatzopoulos.pdf) (https://arxiv.org/abs/2106.09991)
    - Generally for RISC-V arch research and new RISC-V extensions and the like, look at the RISC-V workshop website: https://carrv.github.io/2022/
    - Fast and Accurate Performance Evaluation for RISC-V using Virtual Prototypes (https://www.informatik.uni-bremen.de/agra/doc/konf/2020DATE_Fast_and_Accurate_Performance_Evaluation_RISC-V_VPs.pdf)
    - Validating gem5’s Memory Components - gem5 @ ISCA 22 (https://arch.cs.ucdavis.edu/memory/2022/12/13/validating-memory.html)

- Perf models
    - https://github.com/riscv-software-src/riscv-perf-model
