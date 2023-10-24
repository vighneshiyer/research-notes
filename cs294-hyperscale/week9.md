## SoftSKU (ISCA 2019)

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

There is a tension between web microservices that have diverse requirements (in CPU cores, memory/disk capacity / hierarchy / bandwidth / latency, network, specialized accelerators) and practical datacenter purchasing and maintenance preferences (bulk purchase benefits, resource fungibility, test overheads) towards uniform node configurations.
Since purchasing specialized SKUs for each microservice is impractical, we should leverage the BIOS/kernel-configurable motherboard/core-level knobs to specialize the runtime configuration of a fixed SKU for each microservice.
Facebook has 7 key microservices that can be profiled and have special runtime configurations designed for each one.
Facebook can leverage the fact that they aren't a public cloud, so their microservices run on dedi baremetal servers without virtualization.
7 specialization knobs are leveraged: 1) core frequency, 2) uncore freq, 3) active core count, 4) code v data prioritization in LLC ways, 5) hw prefetcher config, 6) use of transparent huge pages, 7) use of static huge pages.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose uSKU, which automatically varies the configuration knobs at runtime and uses fine-grained profiling to detect small performance diffs - they demonstrate that softSKU achieves 7.2% and 4.5% perf uplifts over the stock and production server configs.
They leverage Intel Resource Director Technology (RDT) to tune LLC runtime knobs.
They collect processor performance data using Operational Data Store (ODS) from 7 microservices (similar to Google-Wide-Profiling).
They make similar observations to prior datacenter profiling works that indicate wide diversity in the performance characteristics and bottlenecks of different microservices.
For instance, compute bound workloads would benefit from improved instruction throughput, but frequently blocked workloads would benefit from greater concurrency, faster thread switching, and better I/O performance.

#### Summarize the weaknesses of the paper. length: one paragraph.

Context switching penalty calculation is iffy - based on extrapolation, not on actual CPU trace data.
They measure performance in uSKU with MIPS via EMON, but that isn't necessarily correlated with actual application throughput - why can't they use a better profiling technique?
Did they also consider cost when varying runtime parameters? In terms of thermal output and potential savings on cooling? Or on the longevity of the throttled machines / disks / memory?
The core frequency and count adjustments seem worthless - seems like we should always operate at max capacity and frequency.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

What is the theoretical benefit of specialized SKUs and how far off are these optimization knob tweaks from getting there?
What is the datacenter losing from loss of SKU specialization? Of course, the cost and complexity tradeoffs don't seem worth specialization, but how far off are we?
In general this motivates more runtime configurability of processors and peripherals based on application profiles (preemptively power gating unused blocks, changing caching strategies).
    - runtime tradeoff of branch predictor data structures and latency?

### Presentation

- Note the Figure 1 - the scales here are stupid, why not show a relative plot for each column? Metrics that have completely different units are plotted together.
- Note Figure 5 - they say workload have vastly different instr mixes, but not really lol
- interesting note that the instr mix of cache1/2 isn't that load/store heavy unlike what others have suggested - substantial effort is spent marshalling data and various arithmetic/control aux things (but those are taxes right?)
- SPEC IPCs are too high, not realistic of actual ILP available in cloud applications
    - Also SPEC front-end stalls aren't that common compared to cloud workloads
    - Google's services are much more backend-bound than Facebook's. Why? Also Facebook services have more IPC variation?
    - SMT is effective since no more than 1/2 of the CPU retire capacity is actually used
    - Meta claims that front-end stalls come from large code footprint (e.g. Web) or frequent context switch and OS activity (e.g. Cache) -> more icache + iTLB entries
    - branch mispreds come from large inst footprint causing aliasing in BTB - based on application, may need to buff up BP or make is simpler and lower latency
- Did google's profiling data come from similar technique as Meta's, there might be some systematic differences leading to systematically offset results

## Memory Hierarchy for Web Search (HPCA 2018)

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Cloud search applications are throughput and latency critical and would benefit from tuned SoC architectures suited for their application characteristics.
These applications differ from workloads (e.g. SPEC, academic open-source search applications) that CPUs are commonly evaluated on and tuned for - they exhibit much lower IPC, and much higher L3 load/L2 instr and branch MPKI.

#### Summarize the paper's contributions. length: one paragraph.

The authors examine cache hierarchies for web search applications, demonstrate that current cache hierarchies do not capture data reuse patterns as effectively as desired, and propose a new SoC architecture that outperforms existing ones.
L3 caches capture code and stack access locality perfectly, but have significant capacity misses for index shard and heap accesses.
They observe that the L2 MPKI for instruction accesses is high, but is low for the L3 (all code fits within cache), but L3 data MPKI is high (can't be cached without a large 1 GiB cache).
They use Intel CAT to dynamically change the cache sizes to manipulate miss rates and memory latencies.
They propose a new SoC architecture with more cores vs L3 cache capacity and a eDRAM-based on-package L4 to hold all heap accesses, and demonstrate that it achieves 27+% improvement over the baseline.

#### Summarize the weaknesses of the paper. length: one paragraph.

They built a rough model of IPC vs L3 AMAT/hit rate using regression on data obtained by cache size manipulation via CAT. They make a bunch of other extrapolations and analytical assumptions when designing their proposed SoC.
Their proposed SoC is too specialized to the numbers from this specific workload. It is unclear whether the L3 cache vs core count allocation they made is ideal for other applications. For instance, they size their L3 specifically to hold the instruction set for this specific search workload.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

In general, the authors propose treating different memory access sources differently when it comes to where and how they are cached.
They distinguish code, stack, heap, and shard accesses and propose ways to efficiently utilize cache capacity by caching specific access types differently based on their expected locality and working set size.
Can we design an API that an application or runtime can exploit to decide what levels of cache each memory segment should be cached in or if a segment should be uncached and also what its prefetch pattern ought to be.
