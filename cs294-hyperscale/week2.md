# Papers

## Profiling a Warehouse-Scale Computer, ISCA '15.

https://research.google.com/pubs/archive/44271.pdf

### Notes

- 30% datacenter tax (common building blocks in software stack needed for cloud applications, but do no meaningful user-facing work)
- server workloads have high icache pressure and memory latency is more critical than bandwidth
    - These are characteristics of workloads that run on an VM that does JIT and GC (they are actually very common in all systems now - mobile, desktop, and server)
    - front end core stall = 15-30% of all pipeline slots, 5-10% of cycles starved on instructions
    - SMT doesn't help much, wide OoO cores aren't used efficiently
- hotspots within an application aren't common, but hotspots across applications are common (the datacenter taxes) - RPC, protobuf serialization, compression
- tail latency is critical metric
- very large instruction footprints due to static large binaries
- They sample Ivy Bridge machines, 1 second samples, using perf, on C++ workloads, gather stack traces and performance counters
- No clear killer application, binaries taking up datacenter cycles are diversifyinga
- Taxes: Protobuf management, rpc, hashing, compression, memory allocation, data movement, kernel (scheduling + syscall handling - 5% of cycles, not acceleratable)
- real applications are much more frontend/backend bound than SPEC benchmarks and don't retire as many instructions per clock
- a bigger icache alone isn't enough - that space must be utilized by a good prefetcher or you will still be front end bound - software profile guided prefetching?
    - treat instructions and data differently - see ARM's Neoverse2 talk which talked about this
    - programs are growing in size - front end bounded-ness will be more critical than backend in the future
- ipc in real apps much lower than in spec
    - mostly due to dcache stalls for backend limits (also somewhat ILP limits)
- even wider SMT might be good - the gains from latency hiding exceed the potential loss of cache sharing overheads

### Questions

- Why do they claim that binaries in the 100s of MBs have no obvious hotspots! it seems odd
    - Individual applications show flat profiles! also odd, maybe an issue related to sampling-based profiling
    - How does this jive with the bimodal IPC observation
- Does the L2 cache pollution between inst/data motivate the Ventana design to partition the L2 into i and d caches?

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Real Google-scale datacenter workloads have particular characteristics which motivate research into accelerating them, in contrast to typical workloads used by computer architects to perform benchmarks (SPEC, Cloudsuite, etc.).
Prior to this work, there wasn't an open publication on how datacenter CPU cycles are actually spent and where the bottlenecks are.
Datacenter workloads are diversifying, so it is critcal to identify their common characteristics and build representative benchmark suites.

#### Summarize the paper's contributions. length: one paragraph.

This was a profiling paper which gathered uArch profiles of Google's datacenter C++ workloads using random sampling of machines and random sampling in time.
The findings include:

- Datacenter applications are very large (100s of MBs) and don't have clear hotspots leading to poor icache utilization and are dominated by front-end related stalls
    - Note that these are also characteristics of workloads that run on an VM that does JIT and GC (they are actually very common in all systems now - mobile, desktop, and server)
- Memory latency is more critical than bandwidth and in most cases, memory bandwidth is heavily underutilized (overprovisioned)
- About 30% of CPU cycles are consumed by "datacenter taxes" which are overheads in cloud applications that do no meaningful user-facing work (these include protobuf management, RPCs, hashing, compression, memory allocation, data movement, and kernel related overheads like scheduling)

The recommendations include:

- Increasing the icache size isn't sufficient on its own, but rather more research into icache prefetching is critical. There may be opportunities in further splitting cache hierarchies into instruction and data caches. During Hotchips, in the ARM Neoverse 2 talk, there was a lot of emphasis on doing different things based on instruction vs data fetches from the core.
- Even wider SMT might be useful - the gains from letting another thread hop onto core resources while one is stalled seems to exceed the overheads of cache and resource contention
- The wimpy core hypothesis is wrong - since ILP is bimodal, a CPU needs to be able to handle bursts of computation in between periods of low utilization of FUs or the aggregate IPC will suffer even more.

#### Summarize the weaknesses of the paper. length: one paragraph.

They only profiled C++ workloads which they claimed ate the largest portion of CPU cycles, but there are potentially other taxes from runtimes in other languages (go, Java) that could have been captured.
The profiling techniques try to estimate uArch behaviors based on performance counters and sampled stack traces, but this might be inaccurate - the MICRO 21 paper, "TIP: Time-Proportional Instruction Profiling" seems to suggest this might be the case.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

Perform more detailed profiling of the same workloads using Firesim-like infrastructure on a concrete microarchitecture to validate the estimated uArch metrics from this work.
Sweep the design space of ROB depth, backend width, and mix of FUs to determine an optimal point for Google-like cloud applications.
Investigate whether profile guided instruction prefetching might be a good fit for Google applications - how must the uArch be modified to support instruction prefetch hints?

## Accelerometer: Understanding Acceleration Opportunities for Data Center Overheads at Hyperscale, ASPLOS '20.

https://research.fb.com/wp-content/uploads/2020/03/Accelerometer-Understanding-Acceleration-Opportunities-for-Data-Center-Overheads-at-Hyperscale.pdf

### Questions

- Are facebook microservices much smaller binaries than Google's maybe non-micro binaries?

### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). Length: one paragraph.

Datacenter applications have different profiles when compared with common CPU benchmarks (e.g. SPECcpu2017) that motivate different microarchitectural choices.
There aren't any representative open benchmark suites or open profiling data for datacenter applications.
There is opportunity to specialize cores and accelerators to target datacenter workloads, but there doesn't exist a good analytical model that can estimate the potential performance and latency improvements.

### Summarize the paper's contributions. Length: one paragraph.

The authors profile Facebook's server fleet with sampling techniques and look at calls to leaf functions (e.g. memcpy, malloc).
They profile a mix of applications and count cycles spent in categories of leaf functions and plot the results - the results are simlar to those reported in the Google paper; most of the cycles (20-50%) are spent in orchestration tasks and not core application logic.
They perform a more detailed breakdown of where cycles are spent than the Google paper, but come to largely the same conclusions: existing CPU benchmarks are not representative of datacenter workloads, there are common acceleration opportunities across workloads (hashing, memory movement, compression), and on-paper IPC improvements don't translate to application performance.
The authors build an analytical model for acceleration that considers how a core can be utilized while an asynchronous accelerator request is ongoing, and validate it against real speedups/latency improvements from deployed accelerators.

### Summarize the weaknesses of the paper. Length: one paragraph.

The sampling approach for application profiling is error prone and may not actually reflect where CPU cycles are being spent.
The Accelerometer model may not model the overheads of having multiple requests in flight and the overheads of resynchronizing responses to their respective threads.
Many of the potential optimization opportunities in Table 4 are quite high level and do not motivate hardware acceleration - they suggest that most of the benefits of optimization come from optimized software rather than specialized hardware.

### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. Assume you have access to similar resources as the paper authors. Length: one paragraph.

We can add more profiling capabilities in Firesim to be able to track retired instructions for both user/kernel space software and reprofile the Facebook applications on RTL to validate the IPC and cycle breakdowns.
We can extract microbenchmarks from the Facebook workloads that exhibit similar breakdowns of time spent in different leaf functions, and do some dummy time-consuming stuff to represent the application logic.
We can estimate the cost of introducing new offloads in terms of engineering cost and software adaptation cost and see whether it makes sense instead of adding more general purpose compute to the fleet.

## The Tail at Scale, CACM February 2013 and Attack of the Killer Microseconds, CACM April 2017

- https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/pdf
- https://www.barroso.org/publications/AttackoftheKillerMicroseconds.pdf
