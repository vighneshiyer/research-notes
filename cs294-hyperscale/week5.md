## Dagger

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Small and frequent RPCs are prevalent in modern microservice webapp architectures, but typical software stacks and peripheral NICs have high per-message overheads that result in high tail latency and limited throughput.
The authors make the case that a core-coupled RPC accelerator is viable to bring down tail latency and improve per-core RPC throughput on both synthetic small message workloads and memcached/MICA KVS.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose the very first NIC that is "closely-coupled" to processors over memory interconnect rather than PCIe-attached or directly-connected (direct MMIO/RF access to the NIC's queues).
They offload the entire software RPC stack into their FPGA-platform NIC in order to minimize per-packet overheads of many small RPCs.
They benchmark their RPC accelerator against various software RPC user-space networking libraries and demonstrate higher per-core RPC throughput and lower tail latency.

#### Summarize the weaknesses of the paper. length: one paragraph.

The authors describe a core-coupled RPC accelerator that plugs into the processor's memory system over a NUMA interconnect - what are the practical considerations of doing this for a datacenter motherboard? It seems that this would require substantial redesign and strongly couple the upgrade cycles of cores and NICs (and their physical interconnect technology).
The authors make strong claims about PCIe latency being very critical to overall tail latency and the viability of microservices in general, but PCIe latency must be dwarfed by other latencies at the system-level, right?
It isn't clear the value of connecting the NIC via UPI vs PCIe; where are the raw latency numbers comparing these two (we know that they have a coherent cache on FPGA enabled by UPI which isn't possible with PCIe, but what are the actual benefits over PCIe DMA)? The numbers reported in the paper show the latency isn't much different, but then where is the primary benefit of UPI coming from? Also the latency numbers look wrong - they shouldn't be in the 100s of uS for 64B DMA.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

I would probably perform a more careful profiling of the existing RPC stacks built on existing PCIe-connected NICs for small message lengths in a real application before diving too deep into a UPI-connected NIC.
To what extent does removing the RPC code from the icache alleviate the frontend pressure seen by typical webapps with large application footprints?

## Protocol Buffer Accelerator

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Data serialization and deserialization via language-agnostic schema formats (e.g. Protobuf) are a substantial datacenter tax (5% as previously reported, 9.5% as measured by the paper) encountered by most webapps.
Since the protobuf wire format and in-memory layout are both fixed and do not evolve rapidly, protobuf serdes is a good target for a dedicated core-coupled accelerator.
Google profiling reveals that most protobuf messages aren't to/from RPCs (but also from storage), so NIC-coupled acceleration isn't a good idea.
It also reveals that most protobuf messages are very small (majority under 32 bytes), so near-core acceleration is critical to avoid fixed per-serdes overheads.

#### Summarize the paper's contributions. length: one paragraph.

The authors develop an open-source protobuf benchmark suite whose data schemas and message features (message size and distribution) are modeled after the characteristics seen in the profiled Google fleet.
They also develop an open-source profobuf accelerator implemented as a RoCC peripheral which works with a modified version of the protobuf library.
Using both the benchmark suite and the accelerator's RTL, the authors run an FPGA-accelerated simulation, boot Linux, and evaluate the performance of their accelerator on the benchmark suite over a software protobuf implementation running on a Xeon.
They demonstrate the overheads of core-to-accelerator communication as a function of message size and motivate the value of a tightly core-coupled accelerator for protobuf serdes.

#### Summarize the weaknesses of the paper. length: one paragraph.

The authors could consider the TCO + NRE costs associated with their custom accelerator - a theoretical savings of 3% of fleet wide cycles with a substantial CPU NRE cost may not be worth it compared to just adding 3% more general-purpose compute to the fleet.
The authors could evaluate the e2e impact of their accelerator on an application - of course, this has many practical difficulties such as the application needing to run on a RISC-V architecture.
The protobuf accelerator directly produces/consumes the in-memory C++ representation of protobuf messages - how would an accelerator that directly interacts with memory work with garbage collected languages (e.g. go, Java) that can move around objects?
Is there any way you would change the protobuf on-wire format to make hardware acceleration easier?

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

It would be interesting to see the impact of the accelerator on icache pressure from less protobuf code needing to be fetched - are the protobuf instructions easily prefetchable or does the accelerator improve performance more than the protobuf serdes acceleration numbers would suggest?
A similar study could also be run on the Xeon core to measure the uarch bottlenecks (frontend/backend/memory bound) of the protobuf code - can profile guided optimization that's specific to a set of message types improve performance on Xeon cores?
