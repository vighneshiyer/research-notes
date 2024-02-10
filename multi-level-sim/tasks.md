# Multi-Level Simulation

## Functional Warmup (L1 DCache)

- Vighnesh's job
- [x] Identify cache state in RTL [d:12/9]
  - Dump cache configuration (or read JSON) from Chipyard SoC
  - Figure out mismatches between RTL cache state and stated cache configuration
  - Evaluate why dcache block size doesn't match RTL
- [x] Pipeclean tag array injection with small design [d:12/10]
  - So far, created a python script to emit a tag array that can be read via readmemb
  - Am able to read it and the contents look right after the ways are reversed
  - Now need to validate I can inject it correctly into the mocked tag array Verilog copied from the Chipyard SoC RTL
  - Next: do this in a generate loop
- [x] Pipeclean data array injection [d:1/23/2024]
  - [x] Validate that the memories look the same as before in old chipyard! Generate new waveforms and RTL collateral
  - [x] Fix up the python script to generate tag and data arrays
  - [x] Validate tag array injection again
  - [x] Add code for data array injection
- [x] Add cache construct with parameters object [d:1/28/2024]
- [x] Clean up tag array pretty print w/ metadata [d:1/29/2024]
- [x] Switch up tag array injection to use multiple files for each memory block [d:1/29/2024]
- [x] Dump data arrays with multiple files [d:1/29/2024]
- [x] Unittest data array dumping [d:1/31/2024]
- [x] Add data array injection to pipeclean [d:1/31/2024]
- [x] Add plusarg for the base of the checkpoint directory [d:2/1/2024]
- [x] Attempt to do direct array injection rather than iteration thru rows [d:2/4/2024]
- [x] Clean up MTR code + tests [d:1/31/2024]
- [x] Fix up spike log parsing + tests [d:2/6/2024]
- [x] Fix up type errors [d:2/6/2024]
- [ ] MTR as iterator of cache checkpoints [d:2/6/2024]
- [ ] Add instruction driven dumping to MTR code
- [ ] Add code to perform cache state injection
  - Do it like GPR injection, but it will generate a bunch of code, may not be so performant
  - Write the forcing logic after 'resetting' period is over
- [ ] get fst file from chipyard for kevin (baremetal test + coremark) [d:2/7/2024]

## CoreMark + HyperCompressBench (w/ lz4)

- Dhruv's job
- [ ] Compile CoreMark to baremetal RISCV and evaluate in RTL sim + TidalSim
- [x] Compile HyperCompressBench benchmark files
- [ ] Attempt to compile zstd to baremetal
  - Iffy - not sure if this is possible
- [ ] Attempt to compile lz4 to baremetal
  - Include file to be compressed inside the binary's static segment
  - Avoid any HTIF syscall proxying
  - Pipeclean in x86 and then spike

## Binary-Agnostic Interval Embeddings

- Raghav's job
- [ ] Review prior work and compile list of all potential features they identified
- [ ] Implement some of those features in Python
- [ ] Evaluate against the BBV embedding

## Chisel Annotations for State Mapping + TestHarness Generation

- [ ] Draft annotation
- [ ] ... eventually we need to inject into Saturn (including RVV registers and CSRs)

## uArch Model Validation Methodology

- General validation methodology for uArch trace models
  - Needs to support all the relevant long-lived arch state blocks
    - Caches (with directory and coherency state), prefetchers, branch predictors
  - Need to compare on identical traces for RTL and uArch model and arch model (which actually emits the trace)
  - The results of the internal state between RTL and uArch model must be the same
  - Therefore need a way to generate legal traces that match the schema of the arch model
  - And a way to replay those traces on RTL and uArch model and correlate uArch states
    - Replaying memory traces in RTL might be hard without involving the core (since L1 cache ports might be difficult to manipulate directly)

## Long-Lived uArch State Identification Methodology

- We can do some kind of waveform analysis
- Find the toggle frequency of registers and RAMs
- Which registers are infrequently set? Need to avoid counting registers that are arch state like CSRs
- What state gets refreshed every cycle or so? Can just threshold the toggle frequency and find the states we need uArch trace models for

## BBV Embedding Perf Opt

- [ ] Use a better BBV construction and querying data structure
  - LRU should be based on PC range not the exact PC! Otherwise it is too wasteful.
  - Create a custom data structure with an alternative implementation that's much faster for exclusive queries
  - Make the constructor take a list of ranges with ids, the structure should be immutable
  - Construct a balanced BST
  - Implement a custom LRU cache based on lookups based on the *range* of the leaf element rather than input PC
- On Young-Jin implementation
  - Original throughput of BB extraction: 664k insts/second
  - New throughput: 746k insts/second, clear improvement but marginal
  - We don't expect BB extraction to get much faster, but queries should get faster - currently broken though
  - BB embedding (via bisect): 480 1k intervals/sec
  - BB embedding (via intervaltree + LRU cache): 651 1k intervals/sec
  - SLOWER! But the cache probably helps a lot here for the highly PC localized aha-mont64 bmark
- Table this until we have a viable implementation
  - Right now, there are 'holes' in the embedding indices with the bisection approach
  - I think we need a custom tree data structure that can be built once we have a sorted PC list + information about which PC boundaries actually form intervals (or we should just have a list of ranges)

## Cache and BP Metrics

- [ ] Find nodes responsible for mispredicts
- [ ] Find nodes responsible for L1d miss + L1i miss + L1d hit + L1i hit + L1i/d access attempt
- [ ] Add MPKI metric computation + print
- [ ] Add cache miss rate computation + print

## Better Cluster Identification

- https://news.ycombinator.com/item?id=38976254

> Cluster stability is a good heuristic that should be more well-known:
>
> For a given k:
>
>   for n=30 or 100 or 300 trials:
>     subsample 80% of the points
>     cluster them
>     compute Fowlkes-Mallow score (available in sklearn) of the subset to the original, restricting only to the instances in the subset (otherwise you can't compute it)
>   output the average f-m score
>
> This essentially measure how "stable" the clusters are. The Fowlkes-Mallow score decreases when instances pop over to other clusters in the subset.
>
> If you do this and plot the average score versus k, you'll see a sharp dropoff at some point. That's the maximal plausible k.
>
> edit: Here's code
>
>   def stability(Z, k):
>     kmeans = KMeans(n_clusters=k, n_init="auto")
>     kmeans.fit(Z)
>     scores = []
>     for i in range(100):
>         # Randomly select 80% of the data, with replacement
>         # TODO: without
>         idx = np.random.choice(Z.shape[0], int(Z.shape[0]*0.8))
>         kmeans2 = KMeans(n_clusters=k, n_init="auto")
>         kmeans2.fit(Z[idx])
>
>         # Compare the two clusterings
>         score = fowlkes_mallows_score(kmeans.labels_[idx], kmeans2.labels_)
>         scores.append(score)
>     scores = np.array(scores)
>     return np.mean(scores), np.std(scores)

## Etc Tasks

### Quick

- [ ] Eliminate stdout prints during tidalsim run
  - For each source of stdout/stderr prints, redirect them into a log file
- [ ] Use spike's `--log=<name>` command line flag to dump a log to file without shell redirection
- [ ] Add log files to record wall time for each step of the flow
- [ ] Interpolate / scale golden sim trace to interval length of the tidalsim run
- [ ] Read LiveSim paper again
    - Focus on how they do offline trace clustering
- [ ] Add additional caching hash based on simulator hash
- [ ] Don't regenerate checkpoints if the binary hasn't changed (in gen-ckpt)
- [ ] Add clustering based on interval-based PCA selection
- [ ] Fix 'chosen_for_rtl_sim' being not a good name
  - Generalize the ability to choose multiple samples to run in simulation
  - Extrapolation should take the mean of all chosen samples for the same cluster
- [ ] Fix the basic block embedding
  - The original Simpoint paper also weights each basic block's feature value by the *number of instructions in that basic block*
- [ ] Disk space saving
  - [ ] lz4 spike commit log
  - [ ] lz4 memory elf
- [x] Add detailed warmup argument
- [x] Handle the tail interval for tidalsim
  - When the program length is not a multiple of the interval length, the last sample gets the wrong IPC!
  - This should be a simple fix to just add some more pickled data from the spike log parsing
- [x] Integrate ref trace collection into tidalsim script (and save results in same run directory)
  - Dump a checkpoint at n_insts = 0
  - Inject that checkpoint into RTL sim and measure perf

### Medium

- [ ] Add spike PMP dumping capabilities
- [ ] Build an error model that models the function of the distance of a interval from its representative centroid and the IPC error
- [ ] Take multiple checkpoints per cluster centroid and evaluate their variance + incorporate into error model
- [ ] For every sample, don't just look at its closest cluster, but weight the IPCs of all the neighboring clusters by distance
- [ ] Use closeness of a given interval to all adjacent clusters
  - Don't just take the closest cluster, look at distances to each one and weight their IPCs
- [ ] Figure out an automated interval length selection strategy
  - We should be able to use a fine grained interval to build the embedding table
  - Then we should be able incrementally coarsen it

### Lengthy

- [ ] Formalize a loadarch file schema (maybe JSON)
  - Secondary: migrate to VPI based state injection away from force based injection
- [ ] Rerunning spike to capture checkpoints is too wasteful
  - Spike should preemptively take snapshots and we should reload from those snapshots when possible and advance minimal time
  - Our existing technique won't scale for larger programs

### VPI-Based State Injection

- Potentially faster than force-based state injection and avoids having to DPI between C and SystemVerilog
- Use `vpi_put_value_array` to inject a 2d reg with a C array
- Some more VPI usage examples: https://stackoverflow.com/questions/76734050/how-to-read-memory-value-at-a-specific-location-using-vpi-and-verilator

### Checkpoint Execution Validation

- We need to check that after N instructions of a checkpoint that the arch state in RTL sim matches what we expect from spike
- Defer this until we have the full flow working since we won't know all the parameters for the scripts involved until then
- Ideally we can just dump spike checkpoints (without memory, just loadarch) and diff that with a loadarch generated from RTL sim
- [ ] Script to dump checkpoints of every ISA test with -p variant
- [ ] Run complex -v test w/ atomics
- [ ] Run riscv-tests benchmarks

## Embedding Analysis

- Need to figure out what the variance of the golden IPC is for a given centroid. How predictable is the IPC?
- Right now, I have no intuition about what characteristics of an interval lead to variance of IPC in its representative intervals
- Play with alternative clustering algorithms vs KMeans
- How good is the existing KMeans clustering? Are there program samples that are big outliers?
- Evaluate whitening/standardization of input matrix before clustering
  - https://stats.stackexchange.com/questions/21222/are-mean-normalization-and-feature-scaling-needed-for-k-means-clustering
  - https://stats.stackexchange.com/questions/448490/when-to-normalization-and-standardization
  - Seems like we should attempt to standardize each feature before normalizing
- Explore dimensionality reduction prior to sample clustering
  - This is only really needed when the matrix gets very large for performance reasons
  - It shouldn't improve the accuracy or robustness of the clustering algorithm


---

OLD TASKS

---

## Archived Tasks

### Unicore MTR Cache Model/Reconstruction

- Simplifying assumptions
  - write-back + allocate
  - LRU instead of random
  - not handling external writes

- Generating commit log
  - `spike -l --log=exp --log-commits -p1 --pmpregions=0 --isa=rv64gc -m2147483648:268435456 $RISCV/riscv64-unknown-elf/share/riscv-tests/isa/rv64ui-p-simple`

- [x] Capturing data
  - use spike commit log
    - bytes written can be inferred from mem line
  - [x] mark writes with cycle, written value, writer CPU
  - [x] mark reads with cycle
    - [x] For future OS/pk support
      - If same as last written value, that's all
      - If not same as last written value, external write
        - only external writes will show updated content on read
  - [x] Potential mismatch between request issue and completion

### Spike Cache Model

- [x] Evaluate the spike cache model code
  - Looks like the spike model doesn't hold the data array, only the tag array
  - There seems to be logging functionality in cachesim.cc to print out misses at every modeled cache level
  - But it seems like the better option is to use the memory commit log (`--log-commits`)
  - We can build our own parser / cache model using MTR (@raghav-g13)

- Prior work
  - MTR: https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=1430560
  - Tycho cache simulator: https://pages.cs.wisc.edu/~larus/warts.html#Tycho
  - LiveCache and LiveSim
- Existing cache models
  - gem5 (painful to extract)
  - zsim (painful to build, but trying)
    - Extracting the cache into a separate project with a simple build system might be viable
  - https://github.com/s-kanev/XIOSim/blob/master/xiosim/zesto-cache.cpp (might be reasonable)

- Reuse other work
  - ~~[ ] Ask on gem5 mailing list (@raghav-g13)~~
  - ~~[ ] Try the cache model in [zsim]( https://github.com/s5z/zsim) (@vighnesh.iyer, @raghav-g13)~~
    - It's not worth reusing other work
- [x] Read MTR paper
- [x] Implement basic unicore MTR cache reconstruction

### Rebase

- [x] Rebase on top of Chipyard main
  - Rebase testchipip
  - Rebase spike
- [x] Validate changes made to tidalsim [d:1/18/2024]
  - Everything looks fine as far as IPC prediction goes, but the RTL simulations have gotten 2x slower!
- [x] Setup meeting time [d:1/20/2024]
- [x] Investigate Chipyard simulation speed regression [d:1/19/2024]
- [x] Use VCS profiling to pinpoint perf regression [d:1/21/2024]

### Report

- [x] Prepare outline in paper form [d:12/5]
- [x] Intro: The problem [d:12/12]
- [x] Intro: Our contributions [d:12/12]
- [x] Background: simulators [d:12/12]

- State injection as usual
- Validation of checkpoints injected and the RTL simulation results (arch state) after simulation
- Arch schema instead of using arbitrary loadarch file format
- Dcache/Icache perf metric extraction from RTL simulation
- Initial version of dcache model + functional warmup
- Evaluation of zero-functional-warmup tidalsim with varying interval length and # of clusters on all embench benchmarks
    - IPC error, runtime, MPKI
- Evaluation of Rocket HW parameter DSE
    - Parameters: Dcache/Icache size/split vs the LLC
    - Same perf metrics used for evaluation
    - Compare against RTL simulation for whether we can do DSE on the right trajectory
- Write only about how to validate warmup models
- Also write about building warmup models for branch predictors and prefetchers
- Also write about language-level features (e.g. in Chisel) to assist uArch/arch state mapping
    - Design-for-simulation (injection-friendly RTL design)

### Class Presentation

- [x] Modify ATHLETE slides for class talk [d:12/10]

### Verilator

- This is archived until it gets fixed upstream in Verilator
- [x] Get Verilator working
  - This isn't that easy due to some internal unsupported capability in verilator
  - It can't force and release unpacked arrays due to unimplemented operators
  - Raghav is working on creating a tiny example to report a bug
    - See [Verilator issue #4735](https://github.com/verilator/verilator/issues/4735)

### Data Extraction Cleanup

- [x] Fix up plotting stuff to use plt.step() [d:11/18]
  - https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.step.html
  - Also see https://matplotlib.org/3.4.3/gallery/ticks_and_spines/multiple_yaxis_with_spines.html
  - Plot the IPC error on the second y axis
- [x] Figure out the mismatch in the number of rows for tidalsim vs reference perf [d:11/20]
  - Some of this is due to the bootrom execution in reference perf
  - This causes a instruction fixed offset!
  - To resolve this, we should inject a snapshot at n_insts = 0 for gathering the ref perf trace
  - This should just be another script rather that can slot the results straight into the same runs directory
  - This is mostly caused by the tail of the trace where RTL sim finishes early due to more frequent HTIF tohost polling for the exit syscall
- [x] Skip bootrom run in spike commit log [d:11/29]
  - Right now, the spike commit log contains a bootrom sequence
  - This isn't part of the binary, so it doesn't get captured in the elf-based BB extraction
  - It also is an inconsistency between the spike and RTL commit logs right now, which causes "# of insts committed" divergence
- [x] Skip bootrom run in RTL sim [d:12/1]
  - Create reference sim run target in tidalsim top-level
  - Also modify parsing function in extrapolation to automatically fetch reference results
- [x] Add 2 axes for ipc error and dist to cluster center [d:12/1]
  - [x] Add absolute IPC error plot
  - [x] Add distance to centroid metric
- [x] Fix the spike checkpointing issue for n_checkpoints larger than 16 [d:12/1]
  - OK one issue is that in interactive mode spike steps by 1 inst every time before calling back into htif tohost/fromhost handling
  - I fixed this inside `interactive_run` by allowing spike to step by `INTERLEAVE` when possible so that the tohost proxy behavior exactly matches normal spike
  - BUT, there is another issue that prevents checkpoints from being dumped:
    - spike terminates the simulation after an `interactive_run` when `tohost` indicates exit syscode (which is the case for the last checkpoint we want to capture)
    - So spike dies before we have the chance to execute the final state dumping commands
    - One potential solution is to add a command line option to disable exit via htif and then the only exit possible is via the interactive quit
    - OK - let me try this - need an entry point for htif_args
    - Need to add +suppress-exit for both spike checkpointing and RTL sims (for non-golden cases for checkpoint injection)

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
