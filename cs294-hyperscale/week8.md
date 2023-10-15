## Memory Tiering at Warehouse Scale (Google)

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

DRAM is expensive, so let's try to replace some of it with networked SSDs/Optane to save cost while not degrading performance too much.
Memory capacity is often overprovisioned since a process will die from OOM rather than just be degraded - so there is an opportunity for cost savings without affecting performance.
Applications can't be rewritten, so the top tier of this tiered memory structure should be transparent and directly accessible and cached by the DRAM.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose TMTS, which uses Optane Persistent Memory as the tier above DRAM, and they provision 25% of the total memory capacity of the nodes with Optane devices.
They modify the kernel to enumerate the Optane memory as a NUMA node and they design a kernel daemon that demotes cold pages to tier 2 NUMA while promoting hot pages to tier 1 NUMA.
The authors don't naively promote a touched page in tier 2 memory to tier 1 immediately (like would be done in a page fault scenario), but rather sample the access frequency in a time window to determine if a hot page should be promoted.
They demonstrate that their system delivers lower cost while resulting in only 1-5% IPC degradation and some impact to tail latency, with a 1% overhead for the userspace+kernel daemons.

#### Summarize the weaknesses of the paper. length: one paragraph.

You can't argue with the evaluation of their system, but the authors didn't present the tradeoff of IPC degradation with lower cost Optane memory against the cost of adding more DRAM to their nodes.
It is not obvious that their approach saves on TCO in aggregate since the IPC degradation necessitates allocating more nodes for each application, which may not balance out the savings from the lower cost memory tier.
They didn't compare their technique against other tiered memory approaches such as zswap, which can be controlled by the same userspace and kernel mechanisms.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

It would be useful to push harder on the application-driven cold memory allocation that the authors discussed briefly.
In general, can we leverage application specific information to adapt promotion policies and to preemptively promote pages before they are likely to be accessed?
A profile guided approach to tune the userspace and kernel daemons would be interesting.

## TPP: Transparent Page Placement

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Memory is expensive, CXL enables dynamic memory expansion as a higher tier'ed memory resources with lower per-GB cost, so let's try to offload cold pages from a node onto far memory.
Far memory has different characteristics (latency/bandwidth) than CPU attached DRAM, which the Linux kernel needs to handle better than it does now.
Application-level characterization of memory access patterns (via Chameleon, another tool developed by the authors) shows that 55%+ of an application's allocated memory is idle within any 2 minute interval - so moving those pages to far memory opens more room for hot pages in local DRAM and enables less DRAM provisioning per node.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose TPP for moving hot/cold pages between memory tiers with proactive page demotion and hot page promotion with this mechanism being fully invisible to the application.
Their kernel modifications include using CXL memory for page reclamation instead of swap, performing preemptive reclamation for local pages, improving NUMA balancing for CXL page promotion by adding hysteresis, and allowing applications to send all pages to CXL memory if before only certain pages are promoted (for apps that are light on memory usage).
TPP is a fully kernel-based solution for page migration and it outperforms existing tiered memory systems (including existing support in the kernel for CXL far memory).

#### Summarize the weaknesses of the paper. length: one paragraph.

The authors didn't have access to a real CXL hardware platform, so they had to mock the characteristics of one with a FPGA-based CXL driver attached to a regular x86 CPU.
The authors should have compared performance of their system where a portion of DRAM is accessed via CXL vs a system with the same memory capacity all on local DRAM - their comparison here is a bit odd since they are comparing the performance of TPP vs regular kernel NUMA mechanisms.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

It would be interesting to integrate more application level information to guide page migration policies, simliar to the Google paper.
As a start, it would be neat to write an userspace allocator that was aware (via some kernel mechanism) of where each virtual page actually resides, and for the application to specify preferences for residence during page allocation.
What are the potential benefits of this approach vs analyzing sampled page access statistics like the paper already does?
