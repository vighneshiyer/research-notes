## FireSim

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Existing microarchitectural evaluation platforms for datacenter applications are either throughput-limited (uArch, cycle-level simulators), low fidelity (analytical models), or high cost (hardware emulators).
To evaluate the impact of custom accelerators, new interconnect technologies, and device disaggregation requires a platform that can model datacenters at RTL-level fidelity with low cost and reasonable throughput (~10 MIPS).

#### Summarize the paper's contributions. length: one paragraph.

The authors leverage AWS F1 FPGAs for low-cost and finely-billable (by the hour) FPGAs for scale-out simulation.
They developed a set of FIRRTL passes to transform arbitrary single clock RTL to a cycle-exact FPGA model via host-target decoupling.
They developed a FPGA DRAM model where FPGA RTL handles the timing model and the FPGA-attached DRAM serves as the backing store.
They developed a C++ switch model that routes packets between their modeled nodes (with NICs) that runs as host software.
They deploy their datacenter simulation of 1024 4-core nodes on 100s of FPGAs and reproduce the noted phenomena of thread imbalance in memcached's impact on tail latency.

#### Summarize the weaknesses of the paper. length: one paragraph.

Their datacenter nodes don't look anything like a real datacenter's nodes. Real nodes would have much beefier processors and just a single core would need to be split across 5-10 FPGAs, let alone the 32+ cores we see on a single socket of a real datacenter node.
Their cost and throughput arguments about the benefits of the RTL-level cloud FPGA-accelerated simulation approach break down when you consider that to simulate a real datacenter system with 1024 nodes, it would cost 100x of what they estimate and run at 1/100'th of the speed they run at.
But the bigger questions is: do we really need RTL-level fidelity to reproduce datacenter phenomena and evaluate datacenter accelerators, interconnects, and cores?
The answer is probably no - sampled simulation techniques + simple microarchitectural models are more than sufficient. The authors never gave any evidence that their results couldn't be reproduced with simpler simulation techniques that are also lower cost.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

Using Firesim to extract microbenchmarks from large workloads would be interesting.
It should be possible to instrument the target with uArch metrics that can be analyzed from a side-channel and the host can determine the points at which an architectural snapshot should be taken of a unique execution fragment.
Such microbenchmarks can then be executed on new target designs in RTL simulation to gain some insight about the impact of microarchitectural changes without having to run the entire FPGA build flow.

## Enzian

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Heterogeneous server blades are becoming a thing, but there isn't a hardware platform for researchers to evaluate how to exploit these blades.
Existing CPU/FPGA platforms are built specifically for one application or market and don't allow researchers to evaluate a design space that could involve CPUs/FPGAs larger or more powerful than what's present on the particular platform.

#### Summarize the paper's contributions. length: one paragraph.

Enzian is a hardware platform that models heterogeneous blades, which might appear in datacenters, by coupling a large FPGA with a server-class CPU (ThunderX-1) with a shared cache-coherent memory space.
Their platform is as open as possible, even allowing modifications to the motherboard controller firmware, with detailed power and clock instrumentation, and is heavily overprovisioned for most applications (allowing DSE).
They demonstrate that Enzian can achieve similar performance to existing server-class hardware platforms while being more extensible and hackable.

#### Summarize the weaknesses of the paper. length: one paragraph.

The motivation for this paper is weak - 1) they claim that other platforms make compromises that are cost-minded and not technical in nature, but that is a given and 2) they claim that their board closely couples the CPU with the FPGA via a coherent interconnect (not PCIe), but such off-the-shelf solutions already exist on the market.
They claim 100ns DRAM latency - sounds unreasonable.
They spent considerable effort on getting the ECI coherency interface to work with the Cavium CPU - it seems quite difficult to modify for research purposes - how do they leverage cross-CPU/FPGA coherency in a way that boards with CAPI couldn't already do?
They claim that having access to modify the BMC is valuable from a research perspective, but didn't show why.

Fundamentally, the authors think there is a good reason for these CPU/FPGA blades to proliferate in datacenters vs regular 2-socket CPU blades with no FPGA (or FPGA attached via PCIe) in terms of cost/power efficiency, TCO, and flexibility for specific applications.
They haven't really made a good argument for that, I think.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

The board-level bringup stuff (related to the BMC and its firmware) is the most interesting aspect of Enzian imo.
I agree with the authors that having a formal model of each board-level IC to automatically determine power sequencing and control algorithms would be a good line of research.
How do we model ICs in a functional way, without relying on partial Verilog models or SPICE models or IBIS models?
How can we automatically extract behavior from ICs by placing them in a (physical) testbed with some partial formal model and Verilog model (+ manual extraction of some things from the datasheet). Can we fill in the gaps via automated electrical testing?
