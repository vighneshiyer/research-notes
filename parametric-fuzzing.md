# Parametric Fuzzing / Constrained Random StimGen API

## 1/25/2022, Wed

- Zest fuzzer works to increase missrate over time
    - ~500 seconds to converge to local maxima, already an excellent result

### TODOs

- Vighnesh's TODOs:
    - Find a suitable arch sim (riscv-perf-model, rivet)
    - Give instructions for compiling a RTL RISC-V simulator for Rocket/BOOM

- Analysis of what stimulus features lead to a higher missrate
- Being able to track which pieces of the random bytestream used as a parameter drive decisions (control) vs data
    - Choosing which sequence to generate = control decision
    - Choosing which register / memory address to r/w from = data decision
    - Choosing an immediate = data decision
- Targeting a different feature
    - Can we use architectural (performance) RISC-V simulators: gem5 RISC-V
    - Metrics: branch mispredict rate, number of pipeline flushes, avg number of entries in ROB, IPC, power, aggregate CPI
- Show that bootstraping a fuzzing run from increasingly refined models up to RTL is much better than fuzzing directly on RTL to begin with + show the power of a functional generative randomization API
    - Target: ICCAD 23 (paper deadline: May 15, 2023)
