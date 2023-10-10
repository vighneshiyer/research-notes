# NoC Power Modeling

## Paper Ideas

- Novel contributions:
    - Evaluation on real RTL (vs a NoC arch model) with post-synthesis golden power numbers from Cadence Joules
    - Identification of additional dynamic NoC node features over ORION 3.0
    - Heterogeneous power model where each NoC node can have different number of ingress/egress, input/output ports, buffers, VCs
    - Formal driven trace generation for a complete and compact training dataset
- Target: ISCA with case study
    - Sweep a space of NoC topologies for a fixed SoC architecture
    - Evaluate power vs perf for each of these topologies and find the tradeoffs
    - Evaluate multi-Gemmini applications using ReRoCC
- Animesh's 219C project report: https://www.overleaf.com/project/6452d2b6393d3653c972be4f
- Animesh's Master's thesis: https://www.overleaf.com/project/643f01719ffc8bfc228b641f
- Power eval configs for Constellation: https://github.com/ucb-bar/constellation/blob/power-eval/src/main/scala/test/Configs.scala
- Repo with scripts for reproduction of results: https://github.com/AnimeshAgrawal/noc-power-eval/

## TODO

- [ ] Reproduce all the results
- [ ] Investigate the role of clock period in waveforms (VCDs passed to Joules)
    - Ask Cadence about what Joules expects from the RTL waveform
    - Create simple reproduction of incorrect static power when the timescale doesn't match the clock period of synthesis
- [ ] Pull out the VCD hacking script used for Jasper's VCD output that adjusts the VCD timescale to match RTL waveforms
    - [ ] Get permissions to Animesh's files on `/scratch` on a5
        - See the `jasper` folder with the Jasper VCD timescale hacking script to add to the `noc-power-eval` repo

## Meetings

### 10/9/2023

- We should write a paper
- The variation of model parameters after generating new input matrices seems to be high, but we don't know why and if this phenomena is even reproducible
- Begin by reproducing results in the Master's thesis
    - Will need to set up environment to run RTL sims on `a` machines and Joules on BWRC machines (annoying)
    - May need to adjust scripts to increase parallelism of Joules runs on BWRC machines
