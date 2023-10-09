## Contiguitas

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Physical memory contiguity is critical to reduce the overhead of address translation (by using larger page sizes) caused by TLB misses.
Small, unmovable allocations (networking buffers, slab/filesystem/page tables) make it difficult to allocate large contiguous pages.
Movable allocations are also a problem since moving a page to a different physical memory location has a significant cost: TLB invalidation latency (which scales poorly with number of cores with private TLBs) and memory copying.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose two techniques: 1) separation of movable allocations from unmovable ones into 2 contiguous regions of physical memory with an adjustable boundary and 2) modifications to the LLC to support migration of unmovable pages even while they are in use.
They implement the first technique in the Linux kernel by separating unmovable and movable allocation memory segments and support for resizing the boundary between the two segments.
The second technique is implemented using SST/QEMU/DRAMSim3 for cycle-level modeling - they model a distributed LLC similar to the production Intel SoCs they are using.
The authors demonstrate performance improvements on Meta datacenter workloads and a higher success rate of large page allocation.

#### Summarize the weaknesses of the paper. length: one paragraph.

To what degree is the inability to allocate 1GB huge pages a limitation of the software running on the machine? Managed runtimes (e.g. JVM) can often allocate huge pages for all heap/stack space and perform all program allocations in userspace and can thereby avoid needing many TLB entries.
The complexity of Contiguitas-HW seems quite high - it seems difficult to implement correctly in RTL and deal with the interplay between the LLC and kernel.
The implementation fidelity of Contiguitas-HW is poor

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

How far is the Contiguitas implementation from the theoretical lower bound on page translation overhead?
We can look at memory access traces and the page tables created by the OS and figure out, given an optimal TLB eviction and replacement policy, what is the ideal way the OS should have allocated pages to maximize successful huge page allocations?
It is hard to tell how much improvement is still left on the table from the paper's analysis, which is mostly application performance-driven.

## SW Defined Far Memory in WSC

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

TCO is too high, too much DRAM is wasted on 'cold data', can we move that data to 'far memory' and reclaim more DRAM capacity?
But 'far memory' doesn't exist! We don't want to add new hardware to the datacenter and ways to interact with it.
Software compression algorithms are pretty good and the OS knows which physical memory pages are cold, so let's leverage that.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose using zswap to do software memory compression of cold pages *proactively* (to avoid tail latency explosions).
They use ML to optimize their control plane (the cold age threshold) based on profiling results collected across the fleet.
They demonstrate a 4-5% savings in memory TCO.

#### Summarize the weaknesses of the paper. length: one paragraph.

The authors didn't discuss the tradeoffs of different compression algorithms and their impact on latency and compression ratio - how compressible is most of the cold data?
We would expect that cold data compression would have some tail latency impacts, but their results show that IPC is unaffected - what are the pathological cases that would result in performance degradation?

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

The node agent right now isn't aware of what applications are being run on the node, but rather just the cold memory stats from the kernel agent - could having application specific metrics/events be useful in further tuning the cold age threshold?
Having too high a value for S makes zswap less efficient - can we tune S more aggressively if we have knowledge of the prior behavior of the running application?
