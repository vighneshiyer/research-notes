# Algorithms for Aspects of Multi-Level Simulation

## Basic Block Identification

- Problem: Given a PC, identify which basic block it resides in

- If we have the source
    - gcc thing
    - llvm/clang thing
    - ??? Raghav enumerate the things here
- If we have the binary
    - Ghidra
- If we have the PC + instruction trace
    - [gem5's implementation](https://github.com/gem5/gem5/blob/3157cde32449fea7b0a0ad5e8241481bc6ee76c3/src/cpu/simple/probes/simpoint.cc#L81) of basic block extraction from PCs
    - [Our implementation](https://github.com/euphoric-hardware/tidalsim/blob/main/pc.py) that analyzes the spike commit log

## Interval Selection

- Simpoint
    - Intervals are just sequences of N instructions (usually 1-100M). The entire program trace is composed of Simpoint intervals. uArch statistics are reported for each interval.
- SMARTS
    - The startpoints of intervals are chosen using [reservior sampling](https://en.wikipedia.org/wiki/Reservoir_sampling). The entire program trace is represented with K such intervals of length N instructions. The SMARTS sampling approach will report predicted average uArch statistics across the entire trace.

## Interval Embedding + Clustering

- Features
    - Basic block vectors
        - [Simpoint implementation](https://github.com/hanhwi/SimPoint/tree/master/analysiscode)
        - Can augment with information about the *order* of basic block traversal
    - uArch metrics based on executing fragements of an interval in a perf model
        - Used in LiveSim to embed intervals
    - Instruction mix (% of add, mul, div, FP, sw/lw, etc.)
    - Privilege mode mix (% of time spent in each priv mode)
    - [See Jerry and Tianrui's project](https://docs.google.com/presentation/d/1QKdaBM06-ziJzGkw_fK2zRcsdx7frJp8iA2XI1xNn8A/edit?usp=sharing)
    - [NPS: A Framework for Accurate Program Sampling Using Graph Neural Network](https://arxiv.org/abs/2304.08880)
- Clustering / Similarity analysis
    - k-means with BIC to determine optimal k
