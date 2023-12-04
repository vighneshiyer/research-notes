# High-Level Ideas / Thoughts on Multi-Level Sim

These are just some random notes, don't take them seriously

## Xiangshan's BetaPoint Work

- At some point, these guys seemed to be working on checkpointing / SimPoint support from their functional ISA simulator (NEMU) and arch state injection into RTL simulation (with functional warmup too)
- They called it "BetaPoint"
- There are scant references to it today, but it was mentioned in their MICRO paper
- See this recent PR (Oct 2023) with references to Betapoint: https://github.com/OpenXiangShan/NEMU/pull/199
- There is some stuff here to: https://github.com/xyyy1420/NEMU/tree/adb43f17f93043a6d779faf9755c2365032d4235/include/profiling
- Searching Betapoint on Github gets stuff like this: https://github.com/OpenXiangShan/XiangShan/issues/1471

## Dynamically Refined Hardware Simulation

- A long time ago, I called this idea "dynamically refined hardware simulation"
- Move from isa sim to arch sim to uarch sim to RTL sim dynamically based on simpoint style sampling and differential snapshot analysis
    - Only execute the parts that need to be run in detailed sim
    - See livehd for an example of this flow that only had 2 stages (isa emulator -> uarch simulator)
- Riscv perf models, simpoint style trace extraction for detailed RTL(and below)-level power analysis
- I think the robotics cosim stuff seems to motivate better modeling (at least a "SimPoint for accelerator based applications" type of thing)
- Dhruv is working on RTL vs gem5 perf correlation
    - One problem: no consistent API to extract out-of-band performance metrics from RTL simulation
    - Also: no good benchmarks that can run in RTL sim fast enough
    - Also: no ability to extract many out-of-band metrics in FireSim - this could be a good interlude project
    - Eventual target: sampled simulation (Simpoint style) for performance numbers on large benchmarks (use Jerry's spike checkpointing and RTL reload feature for sampling)
- Incremental and dynamically refined simulation support
- Trace vs cycle-based simulation
    - Keywords: trace based vs cycle level simulation
    - https://nitish2112.github.io/post/event-driven-simulation/
    - https://auriga.com/blog/2020/full-system-simulation/
- Big systems, very complex, have different types of models for each part of the system, can’t simulate low-level stuff, need to perform early DSE and determine which parts can be accelerated, later need to perform uarch DSE and can’t spend time on recompilation and resimulation (FPGAs / emulators are insufficient, and also rely on having RTL for every part of the system).
    - The idea would be to place models at different levels of abstraction based on the fidelity required

## Google's Call to Action

> I am working with a number of folks here at Google to build a research program to develop better CPU architecture for Google applications. I am envisioning it to be a 3-year program with multiple themes - (1) Benchmark development; (2) Infrastructure; (3) Architecture innovations.
>
> (1) Benchmarks: Google workloads are unique and our previous work shows that they are significantly different from open source benchmarks. The goal of the benchmark research theme is to develop ideas / mechanisms to make open source benchmarks more representative of Google workloads in static and dynamic/runtime behaviors. The output of this research is a set of open source benchmarks that represent Google workload behaviors. These benchmarks are employed by the research community to develop new architecture ideas, tune compilers, and libraries that benefit Google workloads.
>
> (2) Tools and Infrastructure: Tools are critical to enable research for future computer systems. However, accuracy, speed and feed limit what we evaluate. For example, an expectation to boot an operating system on a cycle accurate simulator is going to meet with a lot of disappointments. The goal of this theme is to enable research to come up with better tools for architecture research. Ideas include: (i) new and better ways to do simulation / or system evaluation (e.g. use of ML to speed things up); (ii) hybrid approach to allow us to move and down the scope, accuracy and speed.
>
> (3) Architecture: An important outcome of the research is to develop new architecture features that benefit Google workloads. Previous work has identified a number of unique challenges for Google workloads (e.g. large code footprint, large thread count, frequent context switches, sensitivity to memory latency and bandwidth, etc.). The goal of this theme is to quantify the workload challenges as well as identify new ones. We would also like to collect ideas / solutions from researchers about these challenges.

- The refined simulation idea slots into (2). RTL-level evaluation is what is desired.
- Potential direction of multi-level simulation from high-level analytical model -> functional sim like spike -> cycle-accurate sim like gem5 -> RTL sim

## Notes from Discussion on 8/9/2023

- Things Google is interested in
    - Benchmarking (taking an internal workload -> public representative benchmark, not just execution traces)
    - Infrastructure (efficient simulations), don't want trace based simulations (internally they are evaluating gem5, PIM, llvm's perf model)
        - Here, they are interested in bridging the gap between gem5 and RTL simulation (and spike and analytical models) in the RISC-V ecosystem
    - CPU uArch optimization for hyperscale (not accelerator just stuff), things like frontend and memory hierarchy optimization
- In particular, we should propose in the area of
    - Multi-level simulation of large workloads
    - Transfer of simulation state between different ISAs (e.g. checkpoint machine state on real Intel CPUs and transfer to RISC-V uarch simulator)
- I should write about 2 pages and include at least 1 figure

## Joonho's / Seah's feedback

- Focus on fast iteration loop as the main selling point - otherwise we will not be rewriting any models for which we already have RTL, that doesn't make sense
- Why multi-level simulation? It needs more motivation
    - gem5 can help warmup state for RTL simulation to avoid cache priming/refill overhead (also branch predictor state, TLB, and other stuff that's uArch specific that affects performance a lot)
- We must distinguish between non-modeled uArch state/optimization and modeled state/optimization
    - That which is non-modeled can still have PPA estimated, but it will have to be non-parametric usually (on the gem5 side)
    - That which is modeled must have parameters in gem5 that mimic those in RTL simulation

## Old Background Section

> Background and Prior Work
> There exists a gamut of full system simulators that trade off fidelity, latency, throughput, flexibility, and architecture agnostic-ness. The range of simulators include: analytical models (Aladdin), trace-based simulators (???), functional models (spike), cycle-level execution-driven simulators (gem5, PTLsim), interval simulators (Sniper), timed TLM models, RTL (rocket-chip, BOOM), and FPGA-accelerated simulators (Firesim).
>
> Since it is costly to run full workloads even on low-fidelity simulators, sampled simulation techniques (e.g. Simpoint) only run unique fragments of a workload in simulation, and extrapolate the entire trace from those results.
>
> Combining sampled simulation with multiple simulators has shown to be effective in preserving simulation fidelity, having high instruction throughput, and enabling a fast model iteration cycle. Livesim combines a functional ISA model and a execution-driven timing model of a CPU by checkpointing state in low-fidelity simulation and rerunning unique checkpoints on the timing model to produce an execution trace.
>
> While prior work has demonstrated the effectiveness of this approach, there has been scant work on refining simulation down to the RTL level from functional/performance models.

- https://arxiv.org/abs/2304.11219 (LightningSim: Fast and Accurate Trace-Based Simulation for High-Level Synthesis)
