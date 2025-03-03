# Power Modeling

## Trace Generation

### Objective

Given RTL, generate activity

### Approaches

There are two approaches

1. Generating traces from execution-driven simulation
  - Instruction generators
  - Microbenchmarks
  - Sampling of larger workloads
2. Formal

- evaluation of different riscv instgens vs benchmarks on power variation across parameter space

### Instruction Generators

- [riscv-torture](https://github.com/ucb-bar/riscv-torture)
- [riscv-dv](https://github.com/chipsalliance/riscv-dv)
- [force-riscv](https://github.com/openhwgroup/force-riscv)
- [IBM Microprobe](https://github.com/IBM/microprobe/tree/master/targets/riscv/examples) ([slides](https://riscv.org/wp-content/uploads/2017/12/Tue1424-riscv-microprobe-presentation.pdf))
- [riscv-ctg](https://github.com/riscv-software-src/riscv-ctg)
- [TestRIG](https://github.com/CTSRD-CHERI/TestRIG) (will require quite a bit of hacking for it to work with Rocket/BOOM)
- [MicroTESK](https://forge.ispras.ru/projects/microtesk-riscv) (seems fishy)

### Microbenchmarks / Workloads

- [riscv-tests](https://github.com/riscv-software-src/riscv-tests) (ISA tests + microbenchmarks)
- [imperas-riscv-tests](https://github.com/riscv-ovpsim/imperas-riscv-tests)
- [CoreMark](https://github.com/riscv-boom/riscv-coremark) (can only use a small snippet to avoid long runtimes)
- [rvv-bench](https://github.com/camel-cdr/rvv-bench)
- Booting Linux
- [gemmini-rocc-tests](https://github.com/ucb-bar/gemmini-rocc-tests)
- SPEC (can only use small snippets from random/systematic sampling)

## Macromodeling

- evaluation of manual embeddings (uarch events) vs automatic embeddings (clustering of toggle density) on predictability, understand cross correlation between selected features in each case. what things are missing from each model?

### 3/3/2025 for Nayiri

- You can parallelize GL power sim given a RTL waveform, embarassingly parallel, you will just need a cluster
  - Sampling here is kind of iffy to be honest
- Present the workloads you're using before diving into plots
- BBVs for embedding are reasonable for performance prediction, but kind of fishy for power
  - Your results when including cache miss estimates in the embedding bear this out
- Can you just take over what Ella was working on?
- Can you pick intervals to simulate based on maximally different embeddings rather than centroids?
- Embedding vs IPC prediction study alone is interesting and useful. Variable length intervals would be cool too. Interpretability of the model would be useful.
- Bora suggests trying to use this to guide perf vs energy impact of optimizations like loop unrolling (but we should just race to halt perhaps)
  - Jim suggests race to halt is usually the best heuristic
  - Bora mentions sign and magnitude vs 2s complement in DSP pipelines. Algorithmic transformations to save energy - look at papers from the 90s
  - Sophia mentions that bad prefetching can burn energy and marginally improve performance, so... you should improve the prefetching?
