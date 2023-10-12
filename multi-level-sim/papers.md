# Multi-Level Simulation

## **Survey**

Sampled Processor Simulation: A Survey (2008)[ https://www.sciencedirect.com/science/article/pii/S0065245808000041](https://www.sciencedirect.com/science/article/pii/S0065245808000041)

## **SimPoint**

<https://sites.cs.ucsb.edu/~sherwood/pubs/CHAPTER-simpoint.pdf>

- break up program into intervals (slices of N instructions)
- for each interval, construct basic block vector that tracks how many times each basic block is executed
- use k-means clustering to select phases (groups of similar intervals)
- do full simulation of intervals closest to cluster centroids
  - Can parallelize interval execution
- intervals are large, making warm starting less important
- allows early termination of simulation


- This is profile-based sampling

## **UCSC Jose Renau**
 
ESESC: A fast multicore simulator using time-based sampling (HPCA’13) ([paper](https://ieeexplore.ieee.org/document/6522340))

- updated version of SESC (superscalar simulator)
- sampling based on cycles (i.e. wall clock time) rather than instructions
  - in order to correctly determine where interference between threads will occur
- 3 subintervals - memory system warmup (functional), detailed warmup (timing simulation with recording statistics), detailed timing
  - subinterval length is based on cycles
  - IPC of fast-forwarded parts estimated based on weighted average of previous samples (just using most recent sample worked well too)
  - all threads in same subinterval at same time (based on timing prediction)
- also does sampled simulation of power and temperature
  - I guess this is included in the original SESC simulator
- 9 MIPS (vs 500 KIPS timing simulation, 90 MIPS functional simulation)

LiveSim: Going live with microarchitecture simulation (HPCA’16) ([paper](https://users.soe.ucsc.edu/~renau/docs/hpca16.pdf))

- Run emulation & sample arch state -> clustering -> sample checkpoints to run actual simulations
- Accuracy increases over time as more checkpoints are run
  - provide some kind of results within 5 seconds
- Branch pred → 1M insts of warmup, cache → 50M insts
  - to reduce warmup, emulate very large cache, replay memory ops for blocks in cache
- Follows on from SMARTS and uses statistical sampling
  
LiveSim: A Fast Hot Reload Simulator for HDLs (ISPASS’20) ([paper](https://masc.soe.ucsc.edu/docs/ispass20.pdf))

- mostly about the implementation of checkpointing with partial recompilation
- Targeted towards verification of particular program sequences deeper into the program (not targeted towards sampled performance simulation)

## **SMARTS**

SMARTS: Accelerating Microarchitecture Simulation via Rigorous **Statistical Sampling** (ISCA’03) ([paper](https://infoscience.epfl.ch/record/135578/files/isca03_smarts.pdf))

- “systematic” sampling alternates periods of functional/detailed simulation for a fixed number of instructions
- slower than SimPoint but claims to be more accurate

See **SimpleScalar**

## **LiveSim v SimPoint**

- Extends the idea of sampling simulation \[SP] to two level simulation \[LS]
- Uses perf stats (mentions CPI + other stuff maybe?) from a baseline config for clustering (statistical) v. basic block frequency vector (profile-based) \[SP]
  - LS argues that since performance and code signature correlate, it is okay to change the underlying microarch
  - But given that correlation, what's the motivation to not use basic block frequency? \[Looking at the SMARTS paper for answers]

## **SMARTS v SimPoint**

\[SMARTS] Section 5.3 Comparison to SimPoint
- SP may result in arbitrarily high CPI error
- SP does not offer quantifiable confidence in estimates
  - addressed by the SP authors in a follow-on paper between SMARTS and LiveSim
- Some microarchitecture configurations may cause large variations in behavior across different instances of similarly-profiled basic block sequences.
- The intro of this paper also says the following \[17 is one of the SimPoint papers]
  - Current proposals for simulation sampling suffer from several key shortcomings. On the efficiency front, **most proposals sample several orders of magnitude more instructions than are statistically necessary for their stated error** \[7,10,11,12,17]. This inefficiency is often rooted in their **excessively large sampling units**, either to **amortize the overhead of reconstructing microarchitectural state** or to **capture coarse-grain performance variations by brute force**.

## **Faster Warmup**

Minimal Subset Evaluation: Rapid Warm-up for Simulated Hardware State (ICCD’01) ([paper](https://www.cs.virginia.edu/~skadron/Papers/haskins_iccd_sim.pdf))

MRRL

Efficient Sampling Startup for Sampled Processor Simulation (2005) ([paper](https://cseweb.ucsd.edu/~calder/papers/UCSD-CS2004-803.pdf))

BLRL: Accurate and efficient warmup for sampled processor simulation (2005)

MTR: Accelerating Multiprocessor Simulation with a Memory Timestamp Record (ISPASS’05)([paper](https://ieeexplore.ieee.org/document/1430560))
- Used by LiveSim for fast cache warm-up

several more works not listed

## **Sniper**

Sampled Simulation of Multi-Threaded Applications (ISPASS’13) ([paper](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=6557141)) ([slides](https://snipersim.org/documents/presentations/2013-04-22%20ISPASS%20Multi-threaded%20Sampling.pdf))

BarrierPoint: Sampled Simulation of Multi-Threaded Applications (ISPASS’14) ([paper](https://ieeexplore.ieee.org/document/6844456)) 

pFSA (ISWC’15) ([paper](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=7314164)) 

LoopPoint: Checkpoint-driven Sampled Simulation for Multi-threaded Applications (HPCA’22) ([paper](https://ieeexplore.ieee.org/document/9773236))

## **Old Stuff**

Accelerating Architectural Simulation by Parallel Execution of Trace Samples (1994) ([paper](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=323171))

- early approaches: “simulate the first N instructions of each of the benchmark programs for some large value of N”
- unlike older trace sampling work, uses cache simulator to warm start architectural simulator with exact cache state
- arch simulators ran at 10s of KIPS
- random spacing between samples
- runs 0.5% of SPEC92 to predict within 1% of processor runtime
  - however, up to 5% inaccuracy in cache miss rate and up to 50% inaccuracy on individual basic-block frequencies

Cache modeling via trace sampling

- Accurate Low-Cost Methods for Performance Evaluation of Cache Memory Systems (1988)
- A Comparison of Trace-Sampling Techniques for Multi-Megabyte Caches (1991)
