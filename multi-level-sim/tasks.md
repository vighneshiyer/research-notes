# Multi-Level Simulation

## Tasks

### Pipecleaning Spike Checkpointing

- [ ] Compile baremetal embench-iot benchmarks
    - See Chipyard's embench compilation flow which uses htif-nano.specs
- [ ] Figure out how if we can just run pk alone on gem5 and look at commit log
    - pk alone should boot up just fine but die with a help message (which it can't print since gem5 doesn't emulate syscalls via fesvr and the tohost/fromhost memory locations)
    - Jerry: One potential issue with using pk is that snapshotting spike in the middle of some fesvr interaction and then restoring it might not work (if there is latent state in fesvr that also needs to be ported over)

### ELF Fragmentation

- [ ] Statically identify basic blocks from ELF file (in Rust) (@raghavg-13)
  - Input(s)
    - ELF file
  - Output(s)
    - Sorted Mapping between PCs and BB IDs
    - Reverse mapping between BB IDs and start/end PCs


### PC Trace Fragmentation

- [ ] Review the handbook, create a starter project for PC analysis
- [ ] Write PC trace fragmentation script (also in Rust with tests) (@dvaish)
  - Identify basic blocks from a spike trace log using PCs ( + PPNs etc to separate multiple processes)
  - Input(s)
    - Spike trace log
  - Output(s)
    - Sorted Mapping between PCs and BB IDs
    - Reverse mapping between BB IDs and start/end PCs

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

### Perf Metrics Side Channel

- [ ] Simple IPC calculation dumped every X cycles from TestDriver
  - Peek at the RISCV trace port from TestDriver
  - Count clock cycles and retired instructions
  - Print to stdout every X cycles

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
- [ ] Debug state injection [d:10/28]

- [ ] Read LiveSim paper again
    - Focus on how they do offline trace clustering

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
