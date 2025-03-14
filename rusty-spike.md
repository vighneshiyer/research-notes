## Random Notes

- Master simulation, ganged simulation for dv, slave simulation as trace ingester, single inst replay stepper, symbolic execution, many modes we want to support

- Switch to dbt simulator mode, if possoble
- Exact soc modeling, checkpoint and replay, switch simulation modes during runtime
- Ability to transpile into generic isa ir during execution for generic pass writing
- Maybe that should be a part of tracekit instead
- Create a very high perf riscv disassembler that disassembles into rust native structures either based on inst type (r, I, etc) or semantic inst type (arith, mem, etc)
- Leverage simd

- fesvr + io models + everything on the edge needs to work in RTL sim + FPGA emulation / firesim + functional simulation
  - need to make top-level ports explicit, no internal DPIs
- need to do review of Vienna - someone should look into that

## Early Meeting (Fall 2024)

- Ansh and Pramath will do the first RISC-V spike Rust prototype
  - Initially interpret just rv64ui
  - Support memory ops
  - Hand write assembly tests (or just a shim on top of riscv-tests default env - see 151 tests)
  - Get spike diff testing working very first
- Safin: fesvr integration and rewrite side
- Junha: Vienna

## 1/31/2025

Rusty Spike:

- Zicsr extension + some CSRs defined in supervisor spec
  - https://github.com/five-embeddev/riscv-isa-data/blob/master/csr.yaml
- Testing
  - riscv-isa-tests using p env for rv64imac (fd)
  - next: pm (multicore, benign) + pt (timer interrupts) envs
  - next: v (virtual memory / translation)
  - simultaneously: riscv-benchmarks (might need f/d extensions for some of these)
  - next: embench, coremark
  - next: pk + simple userspace binary (might need HTIF host filesystem syscall support)
  - next: booting Linux

RISC-V Rust Benchmarks:

- So far, ported some of the riscv benchmarks to Rust
- Get dhrystone working using gcc14 cross compiler (warning that becomes an error)
  - gcc14 doesn't like dhrystone
  - Makefile: `RISCV_GCC_OPTS ?=  -Wno-implicit-int -Wno-implicit-function-declaration`
- Port towers not 1:1, just use Vec

## 2/7/2025

RISC-V Rusty benchmarks (Connor):

- Need to investigate multithreaded riscv benchmarks
  - Need to make sure they are running on each core simultaneously
  - It seems like crt.S is holding all cores suspended except hart 1 (... not great, let's get this really working in Rust)
  - Vighnesh to check on this with spike
- Embench semantic port
  - https://github.com/ucb-bar/chipyard/blob/main/software/embench/build.sh
    - Use Chipyard tests as reference for compiling embench for RISC-V
    - Modify the embench source to add some minstret counters before and after the test executes
    - Make sure these work in spike
  - Semantically port one of the benchmarks to Rust (`aha-mont64` is one option)
    - Does Montgomery multiplication
    - https://github.com/rust-num (use some crates here, ask GPT)
- Compile `riscv-tests/benchmarks` with clang to avoid gcc-related discrepancies
  - Vighnesh will send a patch to make this work
- Continue to investigate differences between Rust and C versions of `riscv-tests/benchmarks`

FP ISA support (Ansh):

- Working on FP ISA implementation
- Trying to diff `riscv-tests/benchmarks` against spike using commit log
  - Need to patch up the commit log printed by Rusty spike so it correlates perfectly with spike
  - Safin will make sure text diffing works

CSR support (Safin):

- Working on CSR implementation for Rust codegen
- Appears that for any given CSR, the privilege mode required to write it is the same for all bits
  - However, there are fields within a given CSR that have different read/write behaviors
  - These behaviors should be enforced in the implementation of each field update (but initially, just make the entire CSR writable)

ADL (Ansh):

- ADL docs: https://github.com/euphoric-hardware/riscv-functional-sim?tab=readme-ov-file#architectural-description-languages--generated-iss
  - Why design a new ADL? Why not Sail, riscv-unified-db, VADL?
  - All of them have some critical technical deficiency or annoyance (yes, annoyances are a big deal)
- Commit what you have somewhere, and Vighnesh will sketch out the data types you need

## 2/14/2025

FP ISA support and benchmarks hacking (Ansh):

- FP ISA should be in
  - ISA tests for F still need to be verified
- We need a precise programmatic diff between spike and rusty RISC-V log
  - We built a small test program that just prints one character via HTIF
  - Trying to get the HTIF prints working is difficult without knowing where things diverge
- Need to instrument HTIF device with some prints
  - Figure out why the benchmark trace is dying early (seems to be some unimplemented CSR or instruction causing a trap)

RISC-V Rust benchmarks (Connor):

- Still need to figure out RISC-V multicore benchmarks which don't seem to use all cores
- Embench on RISC-V
  - https://chipyard.readthedocs.io/en/stable/Chipyard-Basics/Initial-Repo-Setup.html
  - Use Chipyard installation process
  - `./build-setup.sh --use-lean-conda` (use this special command line option)
  - Then build embench as usual in `software/embench`
- `aha-mont64`: use GPT or rust-num
- `md5sum`: you already used `md5` crate. Just need to use same stimulus as embench and check the outputs against each other.

- Vighnesh
  - Need to upstream clang in riscv-tests/benchmarks (highly doubt this will be accepted upstream)

## 2/21/2025

- Rusty spike
  - Diffing with spike is WIP, some HTIF issues caused by Joonho's chunking changes that need to be fixed. Even ISA tests appear to fail (just in the exit code handling part though).
  - F extension still not tested
- Embench port
  - md5sum benchmark looks good, checked against embench version
  - Other benchmarks that were easy to port are also done, results line up
  - Overall, looking good
  - Some benchmarks like statemate are just garbage and not worth porting, just ignore those
  - At this point, we will just build our own benchmark suite and divorce completely from ports of `riscv-tests/benchmarks` and embench
- Next step: microbenchmarks
  - Port the stdlib data structure (Vec, HashMap, ...) microbenchmarks to baremetal RISC-V
  - Reuse the benchmark code which should have a wide range of representative inputs and function calls
  - Potentially the benchmarking library needs to be hacked to run on a RISC-V baremetal target
- Note: use the blackbox intrinsic
  - Prevent compiler from doing const prop
  - Montgomery multiplication otherwise is fully const propped and no multiplication is done at runtime lol
- Next step: benchmark data extraction
  - For sha256 and AES we should find crates or applications that use these functions
  - Instrument the functions to capture the arguments
  - Before doing any of that, just look at which applications use these first
  - Avoid the embench approach of hardcoding random input stimuli
- ADL Scala elaboration and circuit traversal
  - This part is hard. Elaborating Chisel and traversing the elaborated circuit in Scala isn't really supported right now.
  - The problem is that the flow looks like: Chisel code -> elaboration -> in-memory chisel.firrtl object -> .fir file -> circt -> bunch of passes -> Verilog
  - Might be worth using this: https://github.com/sequencer/zaozi

## 3/14/2025

- Rusty spike (Ansh)
  - Debugging benchmarks
  - Using native FP in Rust isn't doable, can't access flags. Perhaps have to use x86 specific FP primitives, if we want to use native instructions
  - Someone already built a wrapper around softfloat in Rust, but that doesn't build on ARM
  - Used an alternative soft floating point crate to implement F extension for now
  - Trying to debug exiting in `riscv-tests/benchmarks/mm`. Seems to be doing iffy things during `printf` or the `exit` routine
    - OK found! The problem is that the fast exit path for HTIF isn't implemented in Rust fesvr... very stupid (only the exit syscall was implemented)
  - Next steps: get embench working too + rigorous diffing tool vs spike + begin some NEMU-style performance optimizations (and collect baseline performance numbers)
- Rusty benchmarks (Connor)
  - Rust stdlib microbenchmarks
  - test crate only supports Tier 1 targets (not baremetal RISCV), unclear how much work it would be to modify test to get support for our desired target
  - Other option is to manually port the raw code of the benchmarks to a separate baremetal `no_std` project
  - OK we will try this option! See how much work it is.
  - For stimulus for the embench kernels, we can observe the embench stimuli are just stupid. We should just use some sensible stimulus.
  - Next steps: instrument the aes crate, build some application that uses it, then capture the stimulus. Demonstrate this is possible.
