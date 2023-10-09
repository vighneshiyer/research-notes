## ASIC Clouds: Specializing the Datacenter

https://ieeexplore.ieee.org/document/7551392

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Datacenters today are built out of arrays of CPU/GPU machines, but there may be an opportunity to build compute grids out of ASICs too if it can reduce overall TCO.
The authors make the case that, since Bitcoin mining ASIC clouds exist that have a lower TCO than the equivalent CPU/GPU-based cloud, other ASIC cloud variants are also viable.
The authors point out that a typical way an ASIC cloud comes into existence, is that first FPGAs are brought in for acceleration when the volume is low and the economic incentive is minimal, and slowly we ramp through ASIC nodes as the incentive increases (this was the pattern for Bitcoin, at least).
In order for an ASIC cloud to be viable, the application it targets must be massively parallelizable - either using request-level parallelism (video transcoding) or intrinsic parallelism in the algorithm (ML / Bitcoin mining).

#### Summarize the paper's contributions. length: one paragraph.

The authors build a TCO model for datacenters parameterized by ASIC design/fab costs, DRAM systems, mobos, power delivery, etc. and claim to find pareto-optimal TCO points for different application domains (video transcoding, Litecoin, and CNN inference).
The critical parameters of an ASIC into the model are $ / op/s and W / op/s which are translated into power and perf density; the authors also claim that many ASICs can be placed on a single PCB and since they don't have hotspots like CPUs, they can be cooled more efficiently.
The authors also consider voltage scaling's impacts on power/perf density and consider the limitations of voltage scaling for SRAMs.
The model built by the authors is detailed and unique at the rack-level, but they assume a more typical configuration for the rest of the DC.

"Third, the machine also receives optional tips attached to the transaction" - LOL, these "tips" aren't optional if you want to get your transaction added to the next block.

#### Summarize the weaknesses of the paper. length: one paragraph.

This seems to be quite a fundamental datacenter rack modeling paper, even though it is 'specialized' to ASIC grids within each rack - have the authors validated their power/temp/airflow model against a traditional 2x socket rack first, before speculating on a custom PCB design for ASIC grids?
This paper is very interesting since it seems that constraints of ASIC design (ideal mm2, power envelope, voltage scaling) is motivated by the higher-level rack-level/DC analysis - same question: are CPU dies designed in an ideal way based on this analysis - are they too big/small when compared to optimal as per the model in this paper?
The authors do a good job optimizing the rack design, but there aren't enough details from that to draw TCO and profit numbers, especially when considering the demand curve and the obsolete-ness (over time) of the application (e.g. Bitcoin mining) - there are many factors that the authors haven't considered probably because they don't have access to that info.
The authors make a very strong case for ASIC clouds, but we haven't seen many of them emerge - We do see ASIC clouds today in some ways (FPGA VM instances, TPUs, video encoding) but they aren't really ubquitous, are quite specialized, and have a very tiny share of the overall cloud market - what are the remaining blockers (cost, uncertainty, suitable applications, ASIC design latency)?

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

What happens if instead of PCB-level integration of ASICs, we instead do large packages with chiplets of ASICs (package-level integration) and then combine these packages at PCB-level.
The cross-die throughput, latency, and power should be a lot less within a package, and there should be less cross-package communication necessary, so the ideal TCO point might change.
We should also find a way to evaluate the yield of these custom ASICs as a function of their size and process to evaluate which process a given ASIC cloud should utilize.

## A reconfigurable fabric for accelerating large-scale datacenter services

https://ieeexplore.ieee.org/document/6853195

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

General-purpose CPU server performance has stagnated due to power/thermal limits, but specializated hardware is often too inflexible to adapt to new datacenter applications.
FPGAs provide acceleration potential and reconfigurability and combining them with a within-FPGA network makes it possible to aggregate their computing power.
So let's use a grid of FPGAs to accelerate applications that have enough fine-grained parallelism that can be exploited by custom RTL.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose Catapult, a FPGA cluster architecture (2D torus FPGA-to-FPGA interconnect with an FPGA per rack over PCIe) designed to accelerate the ranking function of Bing search.
They designed a custom FPGA shell to interface custom RTL with board-level functions (DRAM, PCIe, SAS, random IOs) and several infrastructure components (link ECC, CRC, continuous bitstream reconfig) to ensure resilience.
They implement Bing's ranking function as a hybrid software / FPGA algorithm, moving the most compute intensive and parallel parts of the workload to the FPGA fabric (the algorithm is partitioned across FPGAs in the same ring).
They claim a 95% throughput improvement and a lower tail latency for the same throughput over a pure software implementation.

#### Summarize the weaknesses of the paper. length: one paragraph.

How does the network deal with an FPGA link being down? Since there is no dynamic routing, a failure in one FPGA will make certain routes impossible unless the routing tables of every FPGA are reconfigured.
Where do the speedups come from? The authors didn't present software profiling results of the existing algorithm and they also didn't analyze at a high-level what the acceleration opportunities were. e.g. the Feature Extraction stage might be suitable for multicore parallelism similar to FPGA parallelism. Why did the authors believe Bing ranking was a good target for FPGA acceleration vs adding more general-purpose compute to the DC?
The actual numbers are quite disappointing: 95% throughput improvement for 10% more power and 30% higher TCO - if these numbers were examined back then, would the Catapult system still have been built?

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

Evaluate a different approach to acceleration with the same TCO by designing a better rack architecture, interconnect, and hardware.
How does it compare in terms of increased power, flexibility, and longevity?
Actually profile the Bing search application instead of randomly building an accelerator for what seems like a mostly serial workload.
How far off are we from the theoretical max speedup (probably not that much, indicating that this is a poor application all-in-all)
