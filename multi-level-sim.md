# Multi-Level Simulation

- ISA simulation -> uArch trace-based models -> RTL simulation
- Old title: `gem5 Hacking, RTL <-> gem5 Correlation, uArch State Transfer`
- [Original proposal to Google](https://docs.google.com/document/d/1ZIl1rExD4e5BkUvhTFgKjWBVJPtYICGU_o3SSJVmypI/edit?usp=sharing)
- [CS294 project proposal](https://docs.google.com/presentation/d/1tmzARnBtCjhgbhEOnKEhFlAWSjmPCOcvpWcT3EGXO6U/edit?usp=sharing)

## Tasks

- [ ] Read LiveSim paper again [d:10/14]
    - Focus on how they do offline trace clustering

### Pipecleaning Spike Checkpointing

- [ ] Compile baremetal embench-iot benchmarks
    - See Chipyard's embench compilation flow which uses htif-nano.specs
- [ ] Figure out how if we can just run pk alone on gem5 and look at commit log
    - pk alone should boot up just fine but die with a help message (which it can't print since gem5 doesn't emulate syscalls via fesvr and the tohost/fromhost memory locations)
    - Jerry: One potential issue with using pk is that snapshotting spike in the middle of some fesvr interaction and then restoring it might not work (if there is latent state in fesvr that also needs to be ported over)

### PC Trace Fragmentation

- [ ] Review the handbook, create a starter project for PC analysis [d:10/14]
- [ ] Write PC trace fragmentation script (also in Rust with tests)

### Spike Top-Level Runtime

- [ ] Rust top-level spike

### RTL Arch State Injection

- [ ] Inject arch state directly in RTL

### Google Proposal

- [x] Write first draft, send to Sophia [d:t]
- [x] Create one diagram [d:8/14]
- [x] Refine draft with better intro and shorter proposal [d:8/15]

### Refining Proposal

> The skeleton of the checkpointing flow should enable a lot of work...
>
>     generating checkpoints with SimPoint
>     generating checkpoints before accelerator kernels
>     checkpoint restore on firesim
>     generating checkpoints for power modelling
- [x] Gather Joonho's questions into QA section [d:8/21]
- [x] Add more of Joonho's questions to QA section [d:8/23]

### gem5 Hacking

- [x] Build RISCV emulator [d:t]
- [x] Build embench benchmarks [d:t]
- [x] Run benchmarks on spike [d:t]
- [x] Modify spike to poll on tohost much more frequently [d:t]
- [x] Run benchmarks on gem5 [d:t]
- [x] Fix repo to submodule gem5 and embench-iot and use Makefile for builds [d:8/25]
- [x] Add Makefile to script runs of spike and gem5 with stat/commit log collection [d:8/25]
- [x] Check if the gem5 results seem reasonable (stats reasonable + inst retire count match) [d:8/25]
    - They seem reasonable, but some of the IPCs are > 1 - which sounds impossible with the basic CPU model
- [x] Investigate how spike writes checkpoints
- [x] Investigate how RTL sim reads spike checkpoints
- [x] Correlate instruction retire log between spike and gem5
    - They match, except for the fact that pk is in the loop in spike, but not gem5
    - Some benchmarks seem to access pages that result in trap, and then pk handles the fault, but in gem5 those sequences are gone (of course - the syscall emulation handles the syscalls instantly)
- [ ] Build the full system mode Python script for gem5
- [ ] Run the embench binary on gem5 FS mode and just look at the commit log
    - it should trap at some point and die
- No more consideration of gem5 as top - it is too much work - at best we can use some trace-based models from gem5 for cache + coherency, branch predictor, prefetcher

## Meeting Notes

### 10/6/2023, CS294 Project Proposal Discussion with Sagar/Krste

- When changing uArch parameters, the state is hard to map from the uArch model to state in RTL sim
    - Krste suggests using a more abstract model rather than a concrete uArch model so it is easy to convert from trace data to uArch state for a given parameterization
        - See "Memory Timestamp Record" (K.C. Barr, H. Pan, M. Zhang, K. Asanovic, Accelerating Multiprocessor Simulation with a Memory Timestamp Record)
        - For branch predictor functional warmup, capture the branch traces and replay them on a branch predictor model
- This technique is hard to make work for multithreaded workloads on multicore systems
    - Thread synchronization, locks/etc are dependent on the speed of execution of each core
- For each major uArch block that needs functional warmup, come up with a story of how to support them across SoCs
    - Pick a particular block to focus on this semester, but don't only solve a narrow problem (e.g. cache warmup)
    - Try to generalize the technique that you would apply for any given uArch block with long-lived state
    - Try to look at a tougher uArch block than caches (but caches are the most impactful wrt functional warmup fidelity)
    - Sagar: RTL prefetchers aren't too complicated (for what we have), creating a uArch model for them seems reasonable
- For the case study
    - Analyze cache capacity splits between L1i/L1d and also investigate the balance of 2-level cache hierarchies on our RTL
    - Try to answer the question of optimiality between unified vs separate I/D L2 caches on long workloads
    - Look at Tycho (Toicho) and other predecessor work on simulating caches with different parameterizations simultaneouly
- On benchmarks
    - Don't use SPEC
    - Try graph benchmarks ([GAP benchmark suite](http://gap.cs.berkeley.edu/benchmark.html))
    - Try hyperprotobench (can run under pk, protobuf doesn't actually use pthreads, can compile with linux gcc)
        - It should be instruction fetch bound, can model icache pressure effects
    - Think about relative errors to the number of memory accesses, cache misses
- For the practical considerations
    - Raghav: Intervals in simpoint are still around 1M cycles long!
    - Vighnesh: We can't execute an entire interval in RTL sim, we will have to sample even within that interval
- Figure out split between two of us for tasks and how to rope in outside contributors

### 10/3/2023

- Use Chipyard embench build script flags https://github.com/ucb-bar/chipyard/blob/adebd634b4075473b735a355dd010dc8fef8d6c2/software/embench/build.sh
    - To build baremetal embench binaries
    - Then the spike checkpointing flow into RTL simulation should work fine
    - Still worth figuring out how to make pk simulations reloadable into RTL simulation
- Worth checking if the coremark software (https://github.com/ucb-bar/chipyard/tree/main/software) can still be built baremetal - that should be another good target
- Next immediate todos: build both embench and coremark for baremetal, run them on spike and RTL sim, run spike checkpoint and RTL reload flow and make sure they work
    - Next: run the commit logs on your program fragment segmentation script, check the results
    - Next: actually figure out sampling points and build a spike top-level that can dump arch state from those points

### 9/26/2023

- High priority: fix the memif write issue when loading elf checkpoint into RTL simulation
    - We think this is caused by the underlying array not being large enough to hold the mem.elf file contents (2 GiB)
    - TODO: run valgrind on the RTL simulation binary and send stack trace on Slack
    - We should be able to fix this easily enough
- In the meantime: take https://github.com/riscv-software-src/riscv-tests/tree/master/benchmarks
    - Pipeclean arch checkpointing flow from spike and reload into RTL sim using these baremetal benchmarks (don't require pk)
- Once that works, the next step is to determine which PCs (i.e. inst retirement count) need their state dumped
    - Once that basic script works, then we can create a spike top-level that actually programmatically does this
    - This can be written in C++

> Could you point me to the spike branch/fork/docs about using spike as a library. I have an undergrad trying to use AFL to fuzz spike and it would be nice to have spike compiled to a static library with headers
>
> > This works (and is regressed on) in spike/master. There are no docs for the API, but there is a very simple example here https://github.com/riscv-software-src/riscv-isa-sim/blob/master/ci-tests/test-spike showing how to link against  spike and run it.For single-stepping spike-modeled harts, this https://github.com/ucb-bar/chipyard/blob/main/generators/chipyard/src/main/resources/csrc/cospike.cc is probably the best example. Basically you construct an instance of sim_t with the system configuration, and then call sim_t->get_core[hartid]->step() to single-step it.The spike source is pretty readable

- Example of using spike as a library and single-stepping a hart: https://github.com/ucb-bar/chipyard/blob/main/generators/chipyard/src/main/resources/csrc/spiketile.cc

### 9/19/2023

- Basic block extraction script using PC analysis
    - Currently working: extraction of full trace segmented by blocks
    - Clean up this code
- Pipeclean injection of arch state from spike into RTL sim and run for X number of instructions
    - https://chipyard.readthedocs.io/en/stable/Simulation/Software-RTL-Simulation.html#verilator-open-source
    - Commit log with cycle counts (we already have that from RTL sim) - gives us IPC
    - Add feature to stop simulation after X instructions
        - One possibility is to modify the spike checkpoint to inject a tohost_exit right at the basic block boundary
        - Change the testharness that prints out the commit log messages (right now the testharness hooks into a trace port at the top-level of the chipyard SoC, so we can just monitor trace port for number of commited instructions)
- Spike cache model (or standalone cache model)
- Sketch out how we want to inject spike cache state into RTL simulation
    - We need to manually correlate state for now
    - Let's just get some manual force-ing via the testharness working first
    - Figure out: where is the testharness generated from and how can we modify it?

### 9/5/2023

- Attempting to build gem5 full system simulator
    - https://gem5.googlesource.com/public/gem5-resources/+/HEAD/src/riscv-fs/README.md
    - These instructions are clearly quite stale, they suggest building the riscv glibc cross compiler from source when that has been unnecessary for a long time, fetch the toolchain from here instead
        - https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2023.07.07/riscv64-glibc-ubuntu-20.04-gcc-nightly-2023.07.07-nightly.tar.gz
- A lot more investigation is required - it looks like in fs mode, gem5 requires that the user give a big memory blob that it will begin executing (normally linux kernel with bbl) and a disk image it will use for mocking the contents of disk
    - So if we want to get a very basic `spike pk <binary>` equivalent working, we will need to actually bundle bbl with pk with the binary to execute. Or modify `pk` to fetch the binary from a disk image and then begin executing.
    - First question - can we get pk to execute a binary from a disk image (and not using host tether)?
- Talk to Jerry about this - I have very little clue

- Another thread of work: spike execution fragment extraction via basic block coverage
    - Use spike in "library" mode (using spike like an API)
    - Programmatically interact with spike and extract the PC trace
    - Dynamically analyze the PC trace as its coming in and figure out which basic blocks are executing
    - As a first pass, dump the spike commit log (using `-l`), read that log from Python, parse out the PCs, and segment the instructions into basic blocks that are identified by their PC
        - To validate the results, just look at an asmdump and the labels and make sure they match

### 8/29/2023

- gem5 checkpointing in system emulation mode (does it work?) (what schema does the checkpoint have?)
- Can we translate the spike loadarch checkpoint + mem.elf into the gem5 checkpoint schema?
- If that doesn't work, we can add a small init section to the mem.elf that loads all the arch state manually
    - There may be a blocker here, ask Jerry, maybe this is why we use DMI to load arch state in RTL simulation
- Try building a workload with FireMarshal (https://firemarshal.readthedocs.io/en/latest/)
    - Ideally we will avoid using the gem5 linux build infra
- Investigate whether we can use the RISC-V full system mode in gem5 and use pk similar to how we use it with spike

### 8/22/2023

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

### 5/2/2023

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

### 3/16/2023

- Refer to the gem5 paper below
- Also look at a riscv gem5 evaluation: https://carrv.github.io/2021/papers/CARRV2021_paper_63_Chatzopoulos.pdf
- Also take a look at the gem5 model configuration for a real SiFive board: https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/src/python/gem5/prebuilt/riscvmatched/riscvmatched_core.py#85
- Strober paper: https://dl.acm.org/doi/pdf/10.1145/3007787.3001151
- SimPoint stuff: https://cseweb.ucsd.edu/~calder/simpoint/
- Try to play with gem5 riscv - build a simple in-order core model (that looks like your 151 CPU) and run a riscv binary through it

### 2/7/2023

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