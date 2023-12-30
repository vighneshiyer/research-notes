# Thesis Notes

## Qual Outline Sketch

### First Attempt (~mid-2022)

- Title: Improving the efficiency of constrained random via prediction models
- Topics:
    - invariant mining for auto coverpoint insertion + usage for fault localization
    - naive coverage prediction models / blackbox models
    - constrained random models (input arg based prediction)
    - constrained random models (generator decision + graph based prediction)
    - A language for writing auto-instrumented constrained random generators
    - Maximize RTL simulator throughput with SimCommand
    - coverage extrapolation models
    - Evidence of model feedback effectiveness
    - learning time domain correlations
    - validation coverage models for post-silicon defect hunting
- Main points:
    1. Invariant mining
    2. Verification library
    3. Coverage prediction

### Second Attempt (~early-2023)

- Title: improve efficiency of dv, bug hunting, localization, and verification productivity
- Topics:
    - Spec mining for bug localization and assertion synthesis
    - Verification library for chiseltest, showing how to use a high-level language like Scala for unsurpassed verif productivity vs SystemVerilog
        - 3 problems: performance, automatic stimulus tuning, coverage specification
    - Simcommand - showing how we can make this library fast
    - Simulator independent coverage - showing how we can unify coverage collection across simulators, showing the value of using a hardware IR
    - Automatic coverage tuning
        - fuzzing hardware models before fuzzing RTL (bootstrapping)
        - parameteric fuzzing
        - using a coverage proxy model to improve fuzzing time to closure
        - proposal: coverage extrapolation, automatic cov hit technique, hits in time correlation
- Thesis outline sketch:
    - Motivation of improving efficiency of DV
    - Spec mining for bug localization, and assertion / waypoint synthesis
    - High performance testing APIs in high-level languages using continuations
    - Parametric fuzzing
    - Generator based coverage prediction models

#### Notes on Feedback in Hardware Fuzzers in Prior Work

- On feedback being 'useless' in prior hardware fuzzing literature:
- If feedback is required to just make spike emit higher l1 miss rates, then it is clear that the no-feedback being effective is just an artifact of having very poor generators / success / feedback metrics that are too easy to hit
    - just having this result from the spike fuzzing work is sufficient to motivate additional work here!
- Feedback is useless if the generator can't use it properly
    - Or if the metrics are too easy to hit
    - Or if it causes the stimulus evolution to actually slow down or go down the wrong path

#### Notes on ML for DV

- Performing direct stimulus embedding is removing (generator-level) semantics just to try to learn them from data again
- This worked in LLMs
    - You could have embedded some features from linguistic models on the sentences, for example
- But this wasn't necessary because of 2 things
    - Massive datasets
    - Unsupervised learning (could 're-learn' better embeddings than hand constructed ones)
- This can't map to EDA CAD / verification
    - We need to preserve semantics
    - We need to have minimal datasets to avoid overfitting
    - We can perhaps use **active learning** to avoid collecting large supervised datasets
- Unsupervised learning can work for some things
    - Representation learning of RTL for coverage prediction
    - But this also requires some semantic preservation (e.g. we're not doing a string -> string embedding model, but rather a GNN with graph nodes and edges labeled according to their circuit semantic function)

### Third Attempt (~mid-2023)

- Draft thesis outline
    - **Improving the efficiency of constrained random via prediction models**
    - Invariant mining for auto coverpoint insertion + usage for fault localization + redo of this work
    - Naive coverage prediction models / blackbox models
    - Maximize RTL simulator throughput with SimCommand
    - Constrained random preiction models (input arg based prediction)
    - Constrained random prediction models (generator decision + graph based prediction)
    - A language for writing auto-instrumented constrained random generators (parametric generators)
    - Coverage extrapolation models
    - Evidence of model feedback effectiveness
    - Learning time domain correlations between stimulus, functional simulation, and RTL sim
    - Validation coverage models for post-silicon defect hunting

#### Qual Outline

- A DV environment, how can we make each part more efficient?
    - Feedback metrics (use assertion synthesis via spec mining, firrtl passes for time-domain and cross coverage generation, other uarch metrics, power)
        - Overflow checkers -> cover overflow conditions and make sure they are intended
        - Huffman encoder performs some kind of compression estimation before it does compression, this wasn't done in the hardware, a metadata field is proportional to the size of the file being compressed, if an uncompressible file hits the accelerator, it just compressed it anyways, and the metadata field was not big enough (to handle the case when the compressed file > in size than the uncompressed file).
            - Exposed during software decompression with an assertion failure / error message
            - https://github.com/joey0320/compress-acc/blob/zstd-compress/src/main/scala/HufCompressorController.scala#L80
            - This boils down to asserting that no truncation actually occurs even though we're truncating on purpose (we want to make sure the upper bits are actually zero)
        - Basic block coverage counting of SW decompressor to see unhit areas + automatic fuzzing
    - Input generators (use parametric generators)
    - Mutators and/or tuners (use feedback from parametric stimulus generation, use representation learning of RTL, use coverage prediction models, use bootstrapped fuzzing)
    - Testbench API (SimCommand)
    - Bug inducers (irritators) (see the LogicFuzzer paper)
    - Behavior checkers (???, this is custom to a specific RTL block, perhaps spec mining can come in here too)
- **Clarify**: we have these things (RTL (chisel generator), input generator, assertions, I/O interfaces) - can we automatically build a bughunting DV environment for it
    - Use bug inducers e.g. on backpressure

### Fourth Attempt (Nov 2023)

- Quals are scheduled for Jan 17, 2024, 1-3pm. Date is final, need to submit paperwork and book room.
- Also need to make a small departure from what I anticipated the major topic would be from processor fuzzing to uArch simulation and verification (with the focus on simulation methodology).

- I'm realizing that trying to do processor fuzzing as the primary thing for quals is fraught with many issues:
    - Everyone else in prior work is using old RTL and rediscovering existing bugs, and I'll have to do the same (which doesn't add any real value)
    - No one has proposed a good evaluation methodology and I can't just propose one and get away with it without also comparing to prior work's bug discovery on old RTL
    - There are many other groups working on this now in the security realm and they have teams of 5+ people, recent papers in Usenix Security among others
    - I haven't made much progress on replicating prior work or implementing an instruction generator + fuzzing loop from scratch, not clear if I can do better in bug discovery vs prior work - innovation is incremental
- On the other hand, the multi-level simulation direction:
    - Can nearly integrate piecemeal some verification components
    - Already has a good result and a clear path towards getting more results
    - Has many directions to extend the work without too much effort, but high impact
    - Has zero competitiors working in the area
    - Ability to leverage Google's resources for workloads and benchmarks + a motivation to make the methodology ISA-agnostic with clear industrial value
- So instead of trying to fit multi-level simulation into processor fuzzing stuff, I will reverse it and fit a subset of verification topics into multi-level simulation as the main thesis topic

- Things from my past (that I would like to talk about and integrate), sorted from most important to least:
    - [ ] Recent results in multi-level simulation
    - [ ] SimCommand
    - [ ] Parametric fuzzing + instruction generator and targeting of uArch metrics
    - [ ] Specification mining for bug localization (and later for coverpoint synthesis)
    - [ ] Building verification libraries using chiseltest
    - [ ] riscv-dv coverage prediction experiments

#### Organization of qual talk / thesis

- **Proposed Thesis Title**: A High-Fidelity, High-Throughput, Low-Latency Microarchitectural Simulation with Applications in Microarchitecture Evaluation, Verification, and Power Modeling

- Motivation (3 slides)
    - Why we want fast uArch iteration ("due to the end of Moore's law")
    - What currently limits uArch iteration and evaluation latency, what evaluation platforms exist already?
        - Why is uArch sim insufficient? Accuracy, PPA, throughput. As a lab we have always pushed RTL-first.
    - What does fast uArch iteration and evaluation enable?
- The Big Vision (2 slides)
    - Integrate multiple simulators into one platform to provide high throughput, low latency, high fidelity uArch simulations with strong accuracy guarantees
    - Verification of program interval snapshot, restore, and time advancement + verification of uArch trace-based models
    - Utility of waveform fragments (for coverpoint synthesis and power modeling)
    - **Define the scope of the thesis**: which things will I work on and what results do I expect?
- Walkthrough of background and basic flow (~10 slides)
    - Phase behavior of programs
    - Overview of a generic multi-level simulation flow (as seen in prior work)
        - Program interval extraction + embedding (i.e. Simpoints) / reservoir sampling (i.e. SMARTs)
        - Clustering
        - State checkpointing and reload (in RTL/uArch sim)
        - Performance metric extraction
        - Extrapolation + error analysis
    - Functional warmup
        - Long-lived uArch state and its impact on performance metric accuracy
        - What components typically require warmup? How long must they be warmed up?
    - Detailed warmup
        - What parts of the core can be left to detailed warmup?
        - Why we want to use trace-based uArch models for functional warmup vs full-cycle models
    - Prior work
        - SimPoint + derivative work / SMARTs + derivative work
        - Various uArch simulators (gem5, MARSx86, Multi2Sim, PTLSim, Sniper, [ZSim](https://github.com/s5z/zsim), [SST](https://github.com/orgs/sstsimulator/repositories?type=all)) with sampling support
        - LiveSim (HPCA 16)
        - Xiangshan's abandoned attempt
            - [NEMU ISS with arch checkpointing support + abandoned Simpoint](https://github.com/OpenXiangShan/NEMU)
            - BetaPoint (Characteristic Profiling Tools), NEMU-Func (Fast Microarchitectural Warmup)
        - Point out that no one has attempted RTL-level sampled simulation before
            - Point out the challenges such as sub-sampling intervals, uArch state injection
            - The advantages include using only fast warmup models instead of detailed cycle-level simulation and the robustness to errors in the warmup models / interval embeddings
- Our flow and progress so far (~5 slides)
    - Diagram of our flow and modifications necessary to spike/Chipyard
    - Some analysis of interval embeddings and clustering as a function of interval length
    - Early IPC prediction results + errors based on interval distance to centroid
    - Functional warmup using memory timestamp record and cache reconstruction
    - Validation of injection methodology
- General methodology for dealing with long-lived state
    - Caches are just one specific
    - Identification of long-lived state via waveform analysis
    - Validation of modeling: use SimCommand + parametric generators for trace generation
    - Use simulator independent coverage instrumentation methodology for feedback
- Better interval embeddings (seed from Jerry's and Tianrui's investigation)
    - Leveraging trace-based performance simulators - what can be gained?
    - Can we make embeddings binary-agnostic? See the neural graph model embedding paper.
    - What is the tradeoff of Simpoint-style embeddings vs SMARTs-style reservoir sampling?
- Automatic discovery of uArch state
    - FIRRTL pass based mapping (using manual tagging of arch state in RTL)
    - How can we automatically generate mappings between uArch RTL and arch state?
        - Can we use some formal methodology?
    - For the fcsr (fflags) case, can we use formal to map multiple uArch possibilities to the known arch state we want to recreate?
- Advanced topics
    - Modeling host tethering in simulations
        - Differences between RTL and ISA-level simulation (e.g. with fesvr and htif)
    - Dealing with more complex out-of-order cores
    - Dealing wih address translation
    - Dealing with operating systems / context switches / asynchronous events (timer and external interrupts)
    - Dealing with IO
    - Dealing with multicores and coherent caches
    - Dealing with accelerators
    - ISA/simulator/RTL agnostic abstractions
    - Performance tuning (all the various things that contribute to performance)
- Case studies
    - Coverpoint synthesis
        - Leverage multi level sim for waveform reconstruction
            - For spec mining
            - Show that we need high fidelity to expose interesting uArch behaviors (starting at reset isn't good enough - so this is the benefit of functional warmup)
            - Also since we are doing BBV clustering, we can have some confidence that each of these traces is unique
    - Power trace extraction for model training
        - what does low latency buy us?
        - don't want to get into industry evaluation war vs e.g. zebu/haps platforms
- Multi-abstraction integration
- Unifying RTL sim/FireSim and TidalSim

- Separate the what from the how.
    - What: rapid and accurate uarch eval on axes of perf, power, verification
        - Sub-what: unique characteristics of workload extraction for heterogeneous SoCs
    - How: dynamically refined simulation, unified sampling simulation framework and theory, the "Tidal" part of TidalSim

### Fifth and Final (Dec 2023)

- Date is final Jan 17, 2024, 1-3pm. All paperwork is filed and approved.

#### Summary

##### Why

- Specialized and custom silicon is becoming more ubiquitous in all domains from embedded to datacenter
- To design these new SoCs we need tools that enable agile design iteration for target workloads, considering:
    - New heterogeneous SoCs with complex memory hierarchies and accelerators
    - New workloads with characteristics involving IO heavy components + low latency context switches
- Agile design requires simulation collateral that we can leverage for:
    - Performance estimation / bottleneck analysis / application-level profiling
    - HW parameter DSE
    - Power modeling / power trace reconstruction / PPA optimization
    - Generating verification collateral: coverpoint synthesis / bootstrapping fuzzing
- In general, research in microarchitecture simulation has stagnated
    - But now we have RTL for complex SoCs (Chipyard, ESP, OpenPiton). Can we leverage that instead of using the same old performance models (SST, ZSim, gem5, etc.)?

##### What

- Building a fast, accurate, and low latency microarchitectural simulator, which enables:
- Extraction of unique aspects of real programs running on heterogeneous SoCs with respect to power, performance, and functionality
- Leveraging RTL-level simulation outputs for performance/power estimation, DSE, power model training, and coverpoint synthesis

##### How

- Dynamically refined microarchitectural simulation - a hybrid technique blending ideas from Simpoint (interval embedding for similarity estimation), SMARTs (interval sub-sampling and building confidence intervals), and Dynamic Sampling (interleaving execution of functional and performance simulation with feedback, see: "COTSon: Infrastucture for Full-System Simulation")
    - Unifying vanilla, sampled, and fast-forwarded simulation methodologies under one common framework where each technique is just a variant of a general algorithm
        - Similar idea to how FuzzFactory generalized a framework for domain-specific fuzzing
    - Taking microarchitectural simulation down to the RTL-level to avoid the need to correlate performance models and RTL and to produce RTL-level collateral for downstream applications

#### Qual Slides Outline

**Title**: A Unified Microarchitectural Simulation Framework to Identify and Leverage Unique Aspects of Workloads on Heterogeneous SoCs for Power and Performance Estimation and Verification

The title has too many words right now, it needs to be more succint.

- Motivation (**Why**) (3 slides)
    - Discuss the motivation, what are the problems we want to solve, why are existing solutions inadequate?
    - Why we want fast uArch iteration ("due to the end of Moore's law"). How has the SoC and workload landscape changed?
    - What currently limits uArch iteration and evaluation latency? What evaluation platforms exist already?
        - Why are existing simulators inadequate? Accuracy, PPA, throughput. As a lab we have always pushed RTL-first.
    - What does fast uArch iteration and evaluation enable?
- The Big Vision (**What**) (2 slides)
    - A fast, low latency, low cost, highly accurate microarchitectural simulator which simulates down to the RTL-level
        - Integrate multiple simulators into one platform to provide high throughput, low latency, high fidelity uArch simulations with strong accuracy guarantees
        - Verification of program interval snapshot, restore, and time advancement + verification of uArch trace-based models
        - Capturing unique program fragments leveraging microarchitectural RTL-level feedback
    - The applications
        - Performance/power estimation, DSE, power model training, coverpoint synthesis
    - The scope of the thesis
        - What will I work on? - the simulator framework + verification of demonstrating its utility in each application domain
        - What results do I expect? - much faster iteration loop + lower cost + higher accuracy vs prior work
- Prior Work (5 slides)
    - Existing microarchitectural simulators (gem5, SST, ZSim, PTLSim, Sniper)
    - Phase behavior of programs
    - Sampled simulation techniques (SMARTs, Simpoint + variants)
    - Functional warmup models (L1 cache + LLC, branch predictors, prefetchers)
    - Detailed warmup
    - LiveSim (HPCA 16)
- What's New? (3 slides)
    - What are we attempting that is distinct from prior work, and how does it build on earlier efforts?
    - Why RTL-level? - elimininate modeling error (only sampling and warmup error remains), real PPA numbers, leverage RTL-level collateral (e.g. waveforms)
    - Why is RTL-level sampled simulation viable now? - new SoC frameworks (e.g. Chipyard) have high performance, high quality RTL for every part of an SoC + ease of RTL-level microarchitectural DSE + language-level support for architectural and microarchitectural state mapping in RTL
- Our Work Thus Far (Simpoint-style multi-level simulation) (**How**) (10 slides)
    - Diagram of our flow and modifications necessary to spike/Chipyard
    - Some analysis of interval embeddings and clustering as a function of interval length
    - Functional warmup using memory timestamp record and cache reconstruction
    - Validation of injection methodology with arch state comparisons
    - Early IPC + MPKI prediction results + error modeling + comparison to other SOTA simulation techniques
    - Finally, I will discuss problems with our current flow as a prelude to the next part of the talk
    1. Not general or streaming
        - Hardcoded Simpoint-style sampled simulation
        - Requires multiple passes over the commit trace / multiple runs of spike
    1. Inaccurate at modeling asynchronous / timing-aware events (timer/external interrupts)
    1. Hardcoded for a single RTL design point
    1. Functional warmup models aren't validated against RTL
    1. Not warming up all necessary uarch state + resolving ambiguity in restoring arch state
    1. Interval embeddings are binary-aware (not portable) and not microarchitecture-aware (higher liklihood for errors)
    1. Modeling of I/O (off-chip communication, polling, interrupts) will be inaccurate due to uniform treatment of any memory access
    1. Interaction with core-coupled accelerators isn't properly embedded or checkpointable
- Next Steps (TidalSim) (**How (part 2)**) (10 slides)
    - Addressing all the problems above and reaching new frontiers of sampled simulation
    1. Unification of prior techniques in one unified simulation paradigm + building a purely streaming simulator
        - We have implemented one specific simulator design point, but now we will generalize it
        - We will devise a parameterized simulator that can implement SMARTs, Simpoint, and hybrid-embedding/sampling approaches to simulation
        - Our formalization will
            - Produce output metrics such as cost, runtime, throughput, latency, time granularity, error bounds
            - In terms of variables such as number of dynamic instructions, number of cores, RTL sim throughput, and of course the simulation methodology
    1. Putting the 'Tidal' in TidalSim
        - Handling timer and external interrupts and other asynchronous signals
        - Automatic discovery of optimal interval length (variable embedding-driven interval lengths)
        - Moving back and forth between functional sim and RTL sim in the same run
        - Iterative refinement + error correction
    1. Automated generation of RTL arch + uarch state injection from annotations in an HCL (Chisel)
        - Leverage Chisel aspects to annotate arch + uarch state in Scala
        - Use a FIRRTL compiler pass to *generate* an injection testharness specialized for a DUT's HW parameters
        - Enables HW parameter DSE
    1. Verification of functional warmup
        - Functional warmup models should have similar behavior to their corresponding RTL blocks
        - Given a commit trace, turn it into a sequence of events to feed in to both the warmup model AND the RTL block, then evaluate how quickly the state of the functional model diverges from the RTL block
        - Leverage SimCommand + parameteric generators / fuzzing for targeting uArch metrics
        - Demonstrate automatic tuning of the generator to produce pathological behavior that we can then fix - show validation for different design points
    1. Systematic identification of long-lived state and using formal to set uarch state to produce a desired arch state
        - Use waveform + netlist analysis to identify long-lived uarch state structures in the RTL
            - Prune out state that is updated frequently based on its register enable signal / identify RAM contents that are long lived
        - There are certain architectural state bits that are produced via combinational logic from uarch state bits. To set these arch state bits, we need to know how to set the uarch state bits. We can use formal techniques here.
    1. Better interval embeddings
        - Incorporate microarchitectural features from warmup models and RTL simulation into embedding
        - Do not rely on basic blocks, but rather *features* of the instruction trace (instruction mix, memory access pattern, instruction dependencies, estimation of ease of branch prediction / prefetching)
    1. Integration of I/O models (e.g. DMA, NIC) with timing feedback in TidalSim
        - Augment interval embedding with I/O model state
        - Estimate latency of IO via RTL simulation and propagate effective IPC estimates to functional simulation
    1. Sampled simulation of accelerators and heterogeneous SoCs
        - Augment checkpoints with accelerator arch state
        - Augment interval embedding with arch state of accelerator (e.g. the stride / dataflow configuration bits in Gemmini) and features of the accelerator's ISA extension (e.g. the memory addresses given to Gemmini)
    - Things I won't touch
        - Multicore simulation: this is difficult since we have to reconstruct the coherency state of each core's private caches and the LLC for functional warmup and be able to inject all that state correctly. Furthermore, we have to model the instruction interleaving order between cores, and that has traditionally been difficult and a source of significant error for sampled simulators.
- Applications / Case Studies (**How (part 3)**) (7ish slides)
    - Performance trace reconstruction
    - HW parameter design space exploration (for cache hierarchy sweeps and branch predictors)
    - Power trace reconstruction
    - Application-level profiling from sampled simulation
    - Using TidalSim outputs to train a power macromodel (proxy signal identification + regression model)
    - Coverpoint synthesis
        - For evaluation and benchmarking of state space exploration techniques (CRV, formal, fuzzing) (this is doable)
        - For bughunting (this is a hard and risky bet)
    - Bootstrapping fuzzing
    - I can't cover all of these - I'll probably just pick the 3 ones that have the highest chance of success
- Thesis Outline (3 slides)
    - Each of the problems listed above that we propose to solve TidalSim are a thesis chapter
    - I will also write about my prior work in verification libraries, generators, and fuzzing in the "Verification of Functional Warmup" chapter

#### Random Notes

> - Unifying simulation methodologies under 1 framework
>     - SMARTs, Simpoint, hybrid embedding + sampling, time granularity
>         - mixing random sampling into the flow, combine with interval embedding
>     - Accelerated RTL simulation with warmup
>     - Model parameterizes over all simulation methodologies with functions
>         - Can compute things like cost/runtime/throughput/latency in terms of variables like N, C, nCores, RTL_thrput, etc.
>     - Consider types of metrics each simulation platform can deliver and the time granularity + arch vs uarch state models, different reconstruction functions
>     - Combine mixed sim, embedding, sampling with randomness
>         - Dynamically identify intervals for which the embedding features aren't sufficient to predict performance (can also consider uArch warmup model estimated uarch stats)
>         - Can we build an error model?
>     - Then, how to adapt to difficult situations?
>         - Accelerators, dealing with IO (again timesync is an issue)
>     - Unify offline vs online embedding + clustering
>         - Hierarchical incremental clustering vs offline k-means clustering
>         - Error bounds via CLT?
>     - Granularity of perf trace - this is critical to extract unique/relevant segments
>     - Embedding strategies - binary-agnostic vs inst stream based vs ...
>         - Based on RTL feedback too via samples
>     - Sampling as a spectrum
>         - pure RTL sim: infinite window, unwindowed, max speedup = 0
>         - fixed windows, fast-forwarded: max speedup? limited by number of cores and interval length
>             - can't handle external/timer events properly due to time desync, fidelity issues? + warmup model inconsistency wrt RTL
> - New innovations in sampled simulation
>     - TidalSim flow
>     - Variable length intervals
>     - New types of interval embeddings
>       - Cache access information: read/write access counts, miss counts, evictions in a given basic block
>       - Branch prediction estimates: use infinitely sized BHT with 2-bit state, count whether a given branch was predicted correctly, total mispredicts for this given branch, and total number of times this branch has been executes. only captures the "static" predictability of the branch, this is the most naive predictor so it is pessimistic - actual BP behavior will be quite different, but it shouldn't matter much
>       - Since the number of basic blocks and branches is statically known, the uarch-aware embedding vector is of a fixed length
> - Applications
>     - Perf trace reconstruction
>     - Perf trace analysis via application-level profiling
> - Standalone tasks
>     - Synthesizable state injection
>     - Long-lived uarch state identification (waveform analysis / formal)
>     - Reconstructing uarch state for desired arch state (e.g. fflags) - can we do this automatically?
>     - Generating injection testharness / Chisel mark API for arch state
>     - Figure out how many insts of detailed warmup are required - this is a generalization of long-lived uarch state identification - we can get an idea about which register require how much typical time to be set from the reset state + the frequency of updates wrt to each instruction fetch cycle
>
> - cite papers about errors in arch models (also the gem5 riscv rtl comparison paper) - show that already gem5 is so inaccurate, when it comes to timing effects such as interrupts, external IO, and timers, it will be even more inaccurate! so prior work just can't be trusted especially if it calibrated to a single hardware datapoint, but we have no evidence of its ability to extrapolate and maintain accuracy!

## Google Proposal Paragraph

We propose a microarchitectural simulation methodology that combines functional ISA-level simulation, microarchitectural warmup models, and RTL simulation to produce high-fidelity RTL traces of interesting aspects of long-running workloads. We will use sampled simulation techniques from prior art and extend them with better interval embeddings and the ability to inject state into RTL simulation. Our project will entail creating a simulation framework that is both ISA-agnostic and microarchitecture-agnostic using abstractions over these design specific concerns. The applications for this simulation technique include RTL-level performance modeling and prediction on realistic workloads, design space exploration, accelerator evaluation, trace collection for power model training and test, and specification mining.

## Advice From Others

### Advice from Prof. Bora (12/4/2023)

- Based on my ATHLETE quarterly review talk
- I need more **scientific depth**
    - Talk about what is **hard** here
    1. state mapping problem for multi uarch states that map to same arch state
    2. asynchronous events and timers
    3. other hard 'advanced topics'
    4. better interval embeddings that can run with partial information (during interleaved arch and RTL simulation)
    - Basically just presenting engineering work isn't good enough, need some depth and complexity and advanced techniques
- Show a plot of sweeping interval lengths and clusters and all the embench benchmarks and runtime comparison
    - How do we trade off throughput, latency, and accuracy?
- Demonstrate the actual use cases
    - DSE of cache parameters and impact on performance / power
    - Power model training
    - Bug hunting via arch state comparison after injection (for long SPEC-like workloads that we know fail on BOOM on FPGA)
        - We can use this to debug our methodology itself
    - Coverpoint / uarch event synthesis for verification + power model signal/event selection
- What are you trying to do, why, and how are you doing that - use driving examples
    - Quantify the value of multi-level simulation - don't just say 'agile design'
    - Demonstrate speedup in the context of a particular case study vs the baseline techniques
        - e.g. for the cache DSE how would our approach differ vs other techniques (in terms of latency, throughput, accuracy, productivity)

### Advice from Prof. Chris Batten (11/26/2023)

- At a high level, need to 1) show the value of sampled simulation for RTL specifically vs sampling in existing performance models (e.g. gem5 simpoint) and 2) shouldn't pitch it as just a simulation methodology project, but rather as a simulation framework that unifies a bunch of methodologies and enables useful uArch research (in power, performance, and functionality)
    - **Pitch**: A Unified Framework to Find Unique Aspects of Programs (on heterogeneous SoCs) with respect to power, performance, and functionality
    - The selling point is generally avoiding expensive and wasteful RTL simulations of redundant activity via sampling and *what that enables*
- Sampling and simulation methodologies live on a sliding scale
    - Simpoints are usually very large (~10M instructions) and don't require functional warmup - however they are prone to uncertainty wrt clustering and no error bound
    - SMART-based sampling uses smaller intervals (~100k instructions) and require functional warmup since there are many more of them and they are shorter - however they don't give fine time-granularity of performance metrics, only e.g. an entire trace-level averaged IPC, but they have a CLT based error bound
    - Non-sampled RTL simulation by running parallel RTL simulations seeded by checkpoints taken from functional simulation + warmup (scale-out RTL simulation dispatch seems interesting, if only engineering)
    - These techniques are treated as separate things, but can we unify them? Think back to the idea about dynamically refined simulation. Can we combine the SMARTs and Simpoint methods? What makes RTL special (functional warmup in RTL hasn't been done before)?
        - Think about incremental refinement of RTL simulation where we build confidence about the e.g. IPC trace over time
        - How do we bound errors? How can we build a proxy "confidence" metric? E.g. some interval embedding distance metric + uArch state mismatch quantification
- Language / Chisel angles
    - Making functional warmup and arch state injection easy and doable. What made sampled RTL sims non-viable in the past? State mapping + simulation speed.
    - A Chisel API for marking arch/uArch state + emission of HW parameters for the functional warmup models
- Verification/functionality validation angles
    - Coverpoint synthesis in the context of making fuzzing more effective / have a better evaluation methodology
        - We can synthesize coverpoints as fuzzing is happening, but also using waveforms extracted from sampled simulation
        - The goal is to show that existing fuzzers don't perform well without feedback on these synthesized coverpoints
        - This is an extension of specification mining
    - Use functional warmup to bootstrap fuzzing
        - Fuzzers right now start from the reset state and only have ~1000 instructions to get the RTL into an interesting state
        - We can use functional warmup to seed the arch/uArch state of the RTL from a given point and then mutate/generate the remaining instructions from a given PC
    - Comparing uArch states between the functional warmup model and the full RTL simulation (both against the full RTL sim and the end of a given interval)
        - We expect some mismatch, but there might be some cases where the mismatch is substantial and out of the norm
        - These cases can motivate tuning of the functional model or expose RTL bugs
    - Validation of functional warmup models against RTL
        - These models need to roughly match to make sure our functional warmup is accurate and the performance numbers we get are reliable
        - Leverage SimCommand, parameteric generators and fuzzing, and simulator independent coverage to test all scenarios
- Power angles
    - Leverage interesting waveform snippets for training or testing power macromodels
    - Use Tidalsim flow for power trace generation
    - It might also be interesting to look at specialized embeddings for power trace generation?
    - "Thermal warmup" is an interesting angle where the prior program activity influences the thermal profile of the subsequent intervals
- Performance angles
    - DSE-related stuff - explore HW parameter space or SW input space with regular performance metrics being reported
    - Combining RTL and rough perf models for the parts of an SoC we want to modify or play with
    - Retaining high fidelity without having to correlate RTL/perf simulators
- Advanced topics
    - Don't touch multicore stuff - sampled multicore simulation while modeling accuracte coherency interactions is difficult
    - Accelerators are a much more suitable extension
        - Normally the large amount of arch state associated with an accelerator is a disadvantage
        - But this is an advantage for sampled simulation since functional warmup is unnecessary with accelerators where most of the state is architectural vs microarchitectural
            - This is the case for Gemmini, for example
        - May need to generalize the methodology to treat cores and accelerators uniformly
        - Also may need to specialize interval embeddings with accelerator semantics

### Notes from Qual Orientation in Oct 2022

- Talk about: spec mining, verif library, internship results from nvidia + Apple, constrained random, Simcommand, other ongoing undergrad projects
- Plan for Dec application submission for Jan exam. See my prewritten thesis.md plan (this file) for the end (thesis outline) (and make some sections optional)
- Preview the proposal precisely at the very beginning of the slides (all the basics of HW verif can come later)
    - It is critical to get across what the project is in the beginning - do not wait until the middle!

### Ryan's Thesis Advice

> White the thesis as a tutorial on your topic. (See jrk's thesis or David Biancolin's thesis)
> You will find holes that represents papers/projects you have to complete.
> The qual slides should have a thesis outline at the end vs a timeline which is never realistic and obscures details. Make sure the committee knows what your complete plan is and what you have already written.
> Compile a list of Unsolved problems in RTL verification. (meaning of coverage, bug localization theorems, etc.)

### Rohan's Qual Advice

- 45 minutes, keep intro short, but overarching, talk about everything in the talk in the first few slides, then elaborate
- why do you need better tools, what should be improved, what is bad currently?
- **give a grand proposal, overall vision**
- in the intro itself your problem statement and what you're proposing needs to be obvious
- start with the vision before you go into the details
- the proposal must not sound like a paper, but it needs to sound like a grand vision
- Kevin's qual practice notes: https://docs.google.com/document/d/1CfBaUO28fJhuLWHReMhEXTeCjCKh8dTiQ51L3p_9K8g/edit#heading=h.mx3dma3tf45r
    - cutting: less details, more on high level intuition
    - story needs to be **more streamlined**
    - don't split your talk into papers - what did you learn from earlier papers?

### Bora's Advice

- There are three major components
    - Big vision
    - Dive into specifics of the circumstances
    - Then deliver the 'how'
- No more than 35 slides, make the key point / case in slide 7/8
    - Get to the point early
    - 3Qs: why, what (get to this quickly, by slide 7/8), and how
