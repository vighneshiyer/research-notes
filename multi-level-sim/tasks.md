# Multi-Level Simulation

## Tasks

- [ ] Read LiveSim paper again [d:10/14]
    - Focus on how they do offline trace clustering

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

- [ ] Review the handbook, create a starter project for PC analysis [d:10/14]
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

### RTL Arch State Injection

- [ ] Debug DMI load arch segfault (@vighnesh.iyer)
  - Need this as reference
- [ ] Modify test harness top (@vighnesh.iyer)
  - [ ] force statements
  - [ ] identify RTL node from the top level
    - [ ] now: pick 1 Chipyard design, hardcode
    - [ ] later: automate with FIRRTL pass

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
