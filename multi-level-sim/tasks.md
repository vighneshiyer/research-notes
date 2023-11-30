# Multi-Level Simulation

## Tasks

- [ ] Review MICRO papers on RTL simulation
  - [ ] "Hardware/Software Co-Design"
  - [ ] "Khronos"

### Spike Cache Model

- [x] Evaluate the spike cache model code
  - Looks like the spike model doesn't hold the data array, only the tag array
  - There seems to be logging functionality in cachesim.cc to print out misses at every modeled cache level
  - But it seems like the better option is to use the memory commit log (`--log-commits`)
  - We can build our own parser / cache model using MTR

- Prior work
  - MTR: https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=1430560
  - Tycho cache simulator: https://pages.cs.wisc.edu/~larus/warts.html#Tycho
  - LiveCache and LiveSim

- [ ] Read MTR paper [d:11/18]
- [ ] Implement basic unicore MTR cache reconstruction [d:11/18]
- [ ] Identify cache state in RTL [d:11/18]
- [ ] Dump cache configuration (or read JSON) from Chipyard SoC

- [ ] Add code to perform cache state injection
  - ???, do it like GPR injection, but it will generate a bunch of code, may not be so performant

### uArch Model Validation Methodology

- General validation methodology for uArch trace models
  - Needs to support all the relevant long-lived arch state blocks
    - Caches (with directory and coherency state), prefetchers, branch predictors
  - Need to compare on identical traces for RTL and uArch model and arch model (which actually emits the trace)
  - The results of the internal state between RTL and uArch model must be the same
  - Therefore need a way to generate legal traces that match the schema of the arch model
  - And a way to replay those traces on RTL and uArch model and correlate uArch states
    - Replaying memory traces in RTL might be hard without involving the core (since L1 cache ports might be difficult to manipulate directly)

### Long-Lived uArch State Identification Methodology

- We can do some kind of waveform analysis
- Find the toggle frequency of registers and RAMs
- Which registers are infrequently set? Need to avoid counting registers that are arch state like CSRs
- What state gets refreshed every cycle or so? Can just threshold the toggle frequency and find the states we need uArch trace models for

### Low Priority Tasks

- [ ] Fix up plotting stuff to use plt.step() [d:11/18]
  - https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.step.html
  - Also see https://matplotlib.org/3.4.3/gallery/ticks_and_spines/multiple_yaxis_with_spines.html
  - Plot the IPC error on the second y axis
- [ ] Figure out the mismatch in the number of rows for tidalsim vs reference perf [d:11/20]
  - Some of this is due to the bootrom execution in reference perf
  - This causes a instruction fixed offset!
  - To resolve this, we should inject a snapshot at n_insts = 0 for gathering the ref perf trace
  - This should just be another script rather that can slot the results straight into the same runs directory
- [ ] Skip bootrom run in spike commit log [d:11/29]
  - Right now, the spike commit log contains a bootrom sequence
  - This isn't part of the binary, so it doesn't get captured in the elf-based BB extraction
  - It also is an inconsistency between the spike and RTL commit logs right now, which causes "# of insts committed" divergence
- [ ] Instead of using IntervalTree directly, expose as an interface [d:11/29]
  - LRU should be based on PC range not the exact PC! Otherwise it is too wasteful.
  - Create a custom data structure with an alternative implementation that's much faster for exclusive queries
  - Make the constructor take a list of ranges with ids, the structure should be immutable
  - Construct a balanced BST
  - Implement a custom LRU cache based on lookups based on the *range* of the leaf element rather than input PC

- [ ] Get Verilator working
- [ ] Add spike PMP dumping capabilities
- [ ] Formalize a loadarch file schema (maybe JSON)
  - Secondary: migrate to VPI based state injection away from force based injection

- [ ] Use spike's `--log=<name>` command line flag to dump a log to file without shell redirection
- [ ] Read LiveSim paper again
    - Focus on how they do offline trace clustering
- [ ] Add additional caching hash based on simulator hash
- [ ] Handle the tail interval for tidalsim
  - When the program length is not a multiple of the interval length, the last sample gets the wrong IPC!
  - This should be a simple fix to just add some more pickled data from the spike log parsing
- [ ] Integrate ref trace collection into tidalsim script (and save results in same run directory)
  - Dump a checkpoint at n_insts = 0
  - Inject that checkpoint into RTL sim and measure perf
- [ ] Don't regenerate checkpoints if the binary hasn't changed (in gen-ckpt)
- [ ] Add detailed warmup argument
- [ ] Build an error model that models the function of the distance of a interval from its representative centroid and the IPC error
- [ ] Take multiple checkpoints per cluster centroid and evaluate their variance + incorporate into error model
- [ ] Rerunning spike to capture checkpoints is too wasteful
  - Spike should preemptively take snapshots and we should reload from those snapshots when possible and advance minimal time
  - Our existing technique won't scale for larger programs

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

### Embedding Analysis

- Need to figure out what the variance of the golden IPC is for a given centroid. How predictable is the IPC?
- Right now, I have no intuition about what characteristics of an interval lead to variance of IPC in its representative intervals
- Play with alternative clustering algorithms vs KMeans
- How good is the existing KMeans clustering? Are there program samples that are big outliers?
- Evaluate whitening/standardization of input matrix before clustering
- Explore dimensionality reduction prior to sample clustering
  - This is only really needed when the matrix gets very large for performance reasons
  - It shouldn't improve the accuracy or robustness of the clustering algorithm

### PCA-Based N-Clusters Selection

- [ ] Add clustering based on interval-based PCA selection

### Disk Space Saving

- [ ] lz4 spike commit log
- [ ] lz4 memory elf

---

OLD TASKS

---

## Archived Tasks

### ELF Fragmentation

- [x] Statically identify basic blocks from ELF file (@raghavg-13)
  - Input(s)
    - ELF file
  - Output(s)
    - Sorted Mapping between PCs and BB IDs
    - ~~Reverse mapping between BB IDs and start/end PCs~~

### Phase Clustering

- Input(s)
  - Per-interval BBFVs
- Output(s)
  - Representative Intervals (per phase)

- [x] Discuss alternative approaches
  - SMARTS v. SimPoint
- [ ] Do following in Python
  - [ ] Dimensionality Reduction
  - [x] k-means
  - [x] Representative Interval

### Interval Basic Block Frequency Vector Construction

- [x] Run spike, fill BBFV, normalize
  - Input(s)
    - Sorted Mapping between PCs and BB IDs
      - Consider using BST
    - Reverse mapping between BB IDs and start/end PCs
  - Output(s)
    - Per-interval BBFVs

### PC Trace Fragmentation

- [x] Review the handbook, create a starter project for PC analysis
- [x] Write PC trace fragmentation script (also in Rust with tests) (@dvaish)
  - Identify basic blocks from a spike trace log using PCs ( + PPNs etc to separate multiple processes)
  - Input(s)
    - Spike trace log
  - Output(s)
    - Sorted Mapping between PCs and BB IDs
    - Reverse mapping between BB IDs and start/end PCs
- [x] Compare the algorithm against the gem5 version [d:11/10]
  - It looks like the gem5 algorithm has some benefits and drawbacks
  - Benefits:
    - Traverses the commit log as its being generated
      - Emits interval embeddings during the traversal
    - Only a single pass thru the program is required (vs our approach with requires 1. dumping the commit log, 2. traversing it once to construct the PC -> basic block id map, 3. traversing it again to perform the BBV embedding, 4. re-executing spike to dump arch checkpoints at the desired sampling points)
  - Drawbacks:
    - Each subsequent interval might have an embedding with different feature lengths (since new basic blocks might be discovered)
    - Can't vary interval length after program execution (since interval length is baked into the commit traversal)
    - Nested basic blocks aren't properly handled
      - If a basic block turns out to not be a basic block (because it has an entry point mid-way thru the block), then the new basic block is simply constructed as a 'new' block even though it should partition the old one into 2 pieces
      - Perhaps this doesn't matter that much

### Tidalsim Scripting

- [x] Move out the logic in tidalsim into small functions [d:11/13]
- [x] Unit test the functions [d:11/13]
- [x] Migrate the functions in Jupyter to Python source + test [d:11/13]
- [x] Integrate plot generation into separate function [d:11/13]

### Arch State Collection Per Representative Interval

- Input(s)
  - Representative Intervals (per phase)
- Output(s)
  - Mapping between Representative Intervals (per phase) and start/end Arch Checkpoints

- [x] Use spike checkpointing API to collect checkpoints for each given interval

### Debug Instret Discrepancy

- It looks like my instret detection signal is too naive and is logging many more retired instructions than actually exist
- It is not consistent with the printed commit log from RTL sim or the spike commit log
- [x] Done, this is a result of the core clock being 2x as slow as the testbench clock

### Instruction-Gran Perf Metrics

- [x] Switch up metric extraction to inst vs cycle gran [d:11/10]
- [x] Update README [d:11/10]

### Python Top-Level

- [x] Refactor intervals.py / pc.py into functions [d:11/8]
- [x] Unit test with small spike log fragments [d:11/8]
- [x] Fix up basic block vector construction fn [d:11/8]
- [x] Add unit tests for BBV interval embedding [d:11/8]
- [x] Create top-level script with argument list [d:11/9]
- [x] Call spike to dump commit log with same args as ckpt gen [d:11/9]
- [x] Use caching library to cache commit log generation [d:11/9]
  - Turns out there isn't a good caching library that can cope here
  - I will just use manual caching, which seems good enough
- [x] Finish up the rest of the flow up til clustering [d:11/11]
- [x] Add clustering based on manual cluster spec [d:11/11]
- [x] Add checkpoint generation [d:11/11]
- [x] Add IPC collection [d:11/11]
- [x] Cache IPC collection [d:11/12]
- [x] Parallelize IPC collection [d:11/11]
- [x] Add reference perf trace collection [d:11/11]
  - Later: do this properly, instead of manually
- [x] Add plotting for IPC trace [d:11/11]
  - Later: script this, for now, use notebook
  - Looks quite odd - why does the RTL sim perf log go on for 4M instructions when the commit log shows otherwise?
- [x] Clean up repo [d:11/11]
  - Delete unused submodules, archive Makefiles
- [x] Fix up naming schemes + refactor gen-ckpt [d:11/11]
  - Things are too messy!
  - [x] Refactor gen_ckpt so it can be used as a library
- [x] Use gen-ckpt as library [d:11/13]

- [x] Handle tail intervals
- [x] Kill spike sim after taking all the necessary checkpoints (should already happen)
- [x] Optimize BBV embedding with LRU cache [d:11/13]

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
