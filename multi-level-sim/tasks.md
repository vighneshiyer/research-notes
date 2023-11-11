# Multi-Level Simulation

## Tasks

- [ ] Review MICRO papers on RTL simulation [d:11/9]
  - [ ] "Hardware/Software Co-Design"
  - [ ] "Khronos"

### ELF Fragmentation

- [ ] Statically identify basic blocks from ELF file (in Rust) (@raghavg-13)
  - Input(s)
    - ELF file
  - Output(s)
    - Sorted Mapping between PCs and BB IDs
    - Reverse mapping between BB IDs and start/end PCs

### PC Trace Fragmentation

- [x] Review the handbook, create a starter project for PC analysis
- [x] Write PC trace fragmentation script (also in Rust with tests) (@dvaish)
  - Identify basic blocks from a spike trace log using PCs ( + PPNs etc to separate multiple processes)
  - Input(s)
    - Spike trace log
  - Output(s)
    - Sorted Mapping between PCs and BB IDs
    - Reverse mapping between BB IDs and start/end PCs
- [ ] Compare the algorithm against the gem5 version [d:11/10]

### Interval Basic Block Frequency Vector Construction

- [ ] Run spike, fill BBFV, normalize (all in Rust) (@raghav-g13)
  - Input(s)
    - Sorted Mapping between PCs and BB IDs
      - Consider using BST
    - Reverse mapping between BB IDs and start/end PCs
  - Output(s)
    - Per-interval BBFVs

### Phase Clustering

- Input(s)
  - Per-interval BBFVs
- Output(s)
  - Representative Intervals (per phase)

- [ ] Discuss alternative approaches
  - SMARTS v. SimPoint
- [ ] Do following in Python (?)
  - [ ] Dimensionality Reduction
  - [ ] k-means
  - [ ] Representative Interval

### Arch State Collection Per Representative Interval

- Input(s)
  - Representative Intervals (per phase)
- Output(s)
  - Mapping between Representative Intervals (per phase) and start/end Arch Checkpoints

- [ ] Use spike checkpointing API to collect checkpoints for each given interval

### Spike Top-Level Runtime

- [ ] multi-checkpoint support
- [ ] check and if reqd expose checkpointing API

### Python Top-Level

- [x] Refactor intervals.py / pc.py into functions [d:11/8]
- [x] Unit test with small spike log fragments [d:11/8]
- [x] Fix up basic block vector construction fn [d:11/8]
- [x] Add unit tests for BBV interval embedding [d:11/8]
- [ ] Create top-level script with argument list [d:11/9]
- [ ] Call spike to dump commit log with same args as ckpt gen [d:11/9]
- [ ] Use caching library to cache commit log generation [d:11/9]

### Low Priority Tasks / Ideas

- [ ] Formalize a loadarch file schema (maybe JSON)
- [ ] Add spike PMP dumping capabilities
- [ ] Read LiveSim paper again
    - Focus on how they do offline trace clustering

#### VPI-Based State Injection

- Potentially faster than force-based state injection and avoids having to DPI between C and SystemVerilog
- Use `vpi_put_value_array` to inject a 2d reg with a C array
- Some more VPI usage examples: https://stackoverflow.com/questions/76734050/how-to-read-memory-value-at-a-specific-location-using-vpi-and-verilator

#### Checkpoint Execution Validation

- We need to check that after N instructions of a checkpoint that the arch state in RTL sim matches what we expect from spike
- Defer this until we have the full flow working since we won't know all the parameters for the scripts involved until then
- Ideally we can just dump spike checkpoints (without memory, just loadarch) and diff that with a loadarch generated from RTL sim
- [ ] Script to dump checkpoints of every ISA test with -p variant
- [ ] Run complex -v test w/ atomics
- [ ] Run riscv-tests benchmarks

### IPC Side Channel

- [x] Reclone Chipyard to validate existing checkpointing script + injection [d:11/8]
- [x] Identify signals related to IPC computation [d:11/8]
  - See Rocket.sv (look at the $fwrite call, look at `_csr_io_time`, `csr_io_trace_0_valid`, `csr_io_trace_0_exception`)
- [x] Simple IPC calculation dumped every X cycles from TestDriver [d:11/8]
  - Peek at the RISCV trace port from TestDriver
  - Count clock cycles and retired instructions
  - Print to stdout every X cycles
- [x] Add plusarg for sampling period [d:11/8]
- [x] Dump IPC stats to file specified by plusarg [d:11/8]
- [x] Add test termination after N cycles [d:11/8]
- [x] Wait for the core reset to fall before sampling [d:11/8]
- [x] Figure out why instret doesn't match max-instructions [d:11/8]
  - Oh this might be because the final stats aren't printed
  - Yeah that's it, added a tail section to perf.log
- [x] Commit and push [d:11/8]

### RTL Arch State Injection

- [x] Debug DMI load arch segfault [@vighnesh.iyer] [d:10/12]
  - [x] Setup dotfiles on a machines
  - [x] Clone and setup chipyard on a machine
    - This is way more painful than it should be
    - Why does env.sh not contain the $RISCV envvar setting?
  - [x] Run hello test on spike and RTL sim
  - [x] Generate checkpoint
  - [x] Reload checkpoint in RTL sim and reproduce bug
  - ~~[ ] Use valgrind to reproduce stack trace~~
  - ~~[ ] Add prints to memif load~~
  - I can't repro the segfault, although restarting sim in spike doesn't seem to work
  - RTL checkpoint restore seems fine to me, test goes until completion
- [x] Understand `generate-ckpt.sh` script [d:10/15]
- [x] Understand current RTL sim arch state injection via DMI [d:10/15]
- [x] Understand how fast loadmem works in contrast to TSI based init [d:10/15]
- [x] Modify TestDriver.v top to inject arch state (@vighnesh.iyer)
- [x] Create new TestDriver.v in Chipyard srcs + Makevar to pull it in + new naming
- [x] Verify that building a sim still works fine
- [x] Attempt to use simple force statement on the PC and verify via waveform
- [x] Get immediate jumping to rv64ui-p-simple working [d:10/23]
- [x] State injection for VCS [d:10/23]
  - Verilator recompiles too many unchanged dependencies
  - [x] Commit this branch
- [x] Get my environment set up on a machines
  - Without using junest
  - Install: fish, neovim, fd, rg, eza
  - tmux session: conda act, fesvr syscall/htif, conda install failure, chipyard top makefiles
- [x] Pull out loadarch logic into header only library [d:10/23]
- [x] Test loadarch in isolation [d:10/25]
- [x] Add DPI hook for the function [d:10/25]
- [x] Verify that existing DTM loadarch logic works fine [d:10/26]
- [x] Find out how to include loadarch.h when STATE_INJECT=1 [d:10/26]
- [x] Add DPI import from SystemVerilog side [d:10/26]
- [x] Verify that loadarch SV is read correctly [d:10/26]
- [x] Add prints from loadarch C side [d:10/26]
- [x] Compare loadarch prints from C and SV sides [d:10/26]
- [x] Need to intercept the +loadarch plusarg from TestDriver.v, call the DPI function, and then inject state via force
- [x] Identify each arch node and inject [d:10/26]
- ~~[ ] later: automate injection nodes with FIRRTL pass~~
- [x] Debug state injection [d:10/28]
  - Oh this is hard, let's first debug the asm tests and make sure they run clean all the time
- [x] Check state injection for hello.riscv
- ~~[ ] Validate state injection for ISA tests [d:10/30]~~
- ~~[ ] Validate state injection for riscv-tests benchmarks [d:10/30]~~
- [x] Reimplement generate-ckpt.sh as is [d:10/31]
- [x] Take a chance to re-configure environment [d:11/3]
  - [x] Change dotfile manager
    - [x] git
    - [x] alacritty
    - [x] bin
    - [x] make bin only managed on local machine
    - [x] dunst (+ only local)
    - [x] firefox (+ only local)
    - [x] i3
    - [x] redshift
    - [x] t480 (Xresources + Xmodmap in nvim)
    - [x] tmux + specialize shell w/ template
    - [x] nvim
    - [x] ssh config
    - [x] fish (+ templated)
    - [x] note files (+ import from articles + setup.sh)
    - [x] create fresh ssh keys
  - [x] Encrypt and store ssh secrets
  - [x] Merge mill and local dotfiles
  - [x] Get `delta` working with `.gitconfigure`
  - [x] Migrate dotfiles to mill
  - [x] Add nvim-treesitter + bindings
  - [x] Add nvim-telescope + bindings
- [x] Add ability to dump many checkpoints in the same spike run (still pending)
  - [x] Add pyproject.toml
  - [x] Get shrinkwrapped tests + binaries
  - [x] Modify dump_spike_checkpoint to do this
  - [x] Modify spike `dump` command to take in a prefix
- [x] Generalize Ckpt for multiple checkpoints [d:11/6]
- [x] Split loadarch file for multiple checkpoints [d:11/6]
- [x] Parallel memory conversion for multiple checkpoints [d:11/6]
- [x] Dump multiple checkpoints for hello [d:11/6]
  - [x] Verify they work
- [x] Add functionality to parallelize and execute RTL sims
- [x] Run -v-simple test

### Pipecleaning Spike Checkpointing

- [x] Compile baremetal embench-iot benchmarks
    - See Chipyard's embench compilation flow which uses htif-nano.specs
- [x] Figure out how if we can just run pk alone on gem5 and look at commit log
    - pk alone should boot up just fine but die with a help message (which it can't print since gem5 doesn't emulate syscalls via fesvr and the tohost/fromhost memory locations)
    - Jerry: One potential issue with using pk is that snapshotting spike in the middle of some fesvr interaction and then restoring it might not work (if there is latent state in fesvr that also needs to be ported over)


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
