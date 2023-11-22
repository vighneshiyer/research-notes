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
    - [ ] Specificion mining for bug localization (and later for coverpoint synthesis)
    - [ ] Building verification libraries using chiseltest
    - [ ] riscv-dv coverage prediction experiments

#### Organization of qual talk / thesis

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

- Using parametric generators to produce validation stimulus for multi level sim like cache state reconstruction
- Leveraging verification infra for validating uarch simulation
- Leveraging simulatior independent coverage for feedback and instrumentation

## Advice From Others

### Notes from Qual Orientation in Oct 2022

- Talk about: spec mining, verif library, internship results from nvidia + Apple, constrained random, Simcommand, other ongoing undergrad projects
- Plan for Dec application submission for Jan exam. See my prewritten thesis.md plan (this file) for the end (thesis outline) (and make some sections optional)
- Preview the proposal precisely at the very beginning of the slides (all the basics of HW verif can come later)
    - It is critical to get across what the project is in the beginning - do not wait until the middle!

### Ryan's Thesis Advice

> White the thesis as a tutorial on your topic. (See jrk's thesis or David Biancolin's thesis)
> You will find holds that represents papers/projects you have to complete.
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
