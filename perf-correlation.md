# Performance Correlation

## 2/7/2022

- Hansung (maybe GPU RTL), Abe (Chipyard various things)
- Baseline understanding of tools
    - Chisel (https://github.com/freechipsproject/chisel-bootcamp (Ch. 1-3), https://github.com/ucb-bar/chisel-tutorial)
    - Chipyard (building RTL, building an RTL simulator - Verilator / VCS, ran a RISC-V binary through it, insn retirement log)
        - https://github.com/ucb-bar/chipyard
        - https://chipyard.readthedocs.io/en/stable/ (look through Ch 1 and 2) - be able to build a Verilator RTL simulator of the default Chipyard SoC config and run some riscv ISA tests through the RTL simulator
    - ISS (inside Chipyard, there is a ISA simulator called spike - spike can also run RISC-V binaries, it will also give you an insn retirement log)
        - https://github.com/riscv-software-src/riscv-isa-sim

- TODOs:
    - Build spike from source (in Chipyard)
    - Run spike on RISC-V binaries (chipyard/toolchains/riscv-tools/riscv-tests)
    - Check the cmdline arguments of spike and get a insn retirement log out
    - Check this against the log from RTL simulation

- Read through this guide, and try compiling gem5 and running x86 binaries through it, and get out a performance trace: https://www.gem5.org/documentation/learning_gem5/introduction/
- Investigate how to get the RISC-V port of gem5 working, and run RISC-V binaries through it with a baseline CPU uArch model (we can just model a simple in-order 5 stage pipeline with the typical bypass paths you're used to)

- Relevant papers:
    - Architectural Simulators Considered Harmful (https://ieeexplore.ieee.org/abstract/document/7155440) - a critique of arch sim, and their accuracy, and why it is better to evaluate RTL, and if you were to build an arch sim - why you should have knowledge of RTL design to begin with
    - gem5 papers:
        - The gem5 simulator: https://dl.acm.org/doi/abs/10.1145/2024716.2024718
        - Micro-architectural simulation of in-order and out-of-order ARM microprocessors with gem5: https://ieeexplore.ieee.org/abstract/document/6893220
    - risc-v gem5 papers:
        - RISC5: Implementing the RISC-V ISA in gem5: https://carrv.github.io/2017/papers/roelke-risc5-carrv2017.pdf
        - Simulating Multi-Core RISC-V Systems in gem5: https://www.csl.cornell.edu/~cbatten/pdfs/ta-gem5-riscv-slides-carrv2018.pdf
        - Towards Accurate Performance Modeling of RISC-V Designs(https://carrv.github.io/2021/slides/CARRV2021_slides_63_Chatzopoulos.pdf) (https://arxiv.org/abs/2106.09991)
    - Generally for RISC-V arch research and new RISC-V extensions and the like, look at the RISC-V workshop website: https://carrv.github.io/2022/
    - Fast and Accurate Performance Evaluation for RISC-V using Virtual Prototypes (https://www.informatik.uni-bremen.de/agra/doc/konf/2020DATE_Fast_and_Accurate_Performance_Evaluation_RISC-V_VPs.pdf)
    - Validating gem5???s Memory Components - gem5 @ ISCA 22 (https://arch.cs.ucdavis.edu/memory/2022/12/13/validating-memory.html)

- Perf models
    - https://github.com/riscv-software-src/riscv-perf-model
