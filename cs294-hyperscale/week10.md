## Cloud-Scale Characterization of RPCs

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

RPCs are increasingly used in cloud workloads - they incur a substantial "latency tax".
They show that RPCs operate at 'ms' timescales (rather than 'us' scale micro RPCs), their latencies on average are dominated by application processing, but the tail latency is dominated by RPC latency variability (due to network).
It is important to understand RPC characteristics so people don't build useless accelerators / NIC architectures to optimize some metric that doesn't matter in practice.

#### Summarize the paper's contributions. length: one paragraph.

The authors profile RPCs in Google internal services using several Google fleet-scale profiling tools. They make the following observations:
- RPC latencies don't go below 100s of uS, and in the tail go up to 100s of ms or even seconds
- A few RPCs are very popular and the most popular ones generally have lower latency
- However, while slow RPCs are only a small fraction of the total # of calls, they take up 89% of the total RPC time
- RPC calls trees are much wider than they are deep
- Most RPC messages are very small (less than 10 KB), but for most methods, there is a tail with much larger message sizes (in the 100s of KB)

#### Summarize the weaknesses of the paper. length: one paragraph.

The paper is quite Google workload specific and the insights are quite broad and uninteresting in nature. Without a representative benchmark suite that academics can leverage to do better RPC acceleration research, there isn't a good point to this paper.
"We conclude that system-level optimizations, ..., may benefit from application specificity". lol, the entire paper is like this - a bunch of obvious 'insights' and a bunch of very specific data that doesn't motivate anything specific.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

The authors could actually implement one optimization that they discussed in section 5 and show the benefit on application throughput / tail latency through mitigating various RPC taxes.
Something that only a large hyperscaler can do is evaluate and optimize physical co-location of services and services they invoke downstream and its impacts on network congestion / latency.

## Profiling Hyperscale Big Data Processing

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Big data is big, and we need hardware accelerators to deal with the bigness.
Modern databases are sharded and replicated and have many caching layers and query APIs - they also use different memory systems (RAM, SSD, HDD), and the authors point out that skewed ratios of SSD:HDD storage might motivate an intermediary caching layer.

#### Summarize the paper's contributions. length: one paragraph.

The authors characterize Spanner, BigTable, and BigQuery and show how the application profile is split among compute, storage, shuffle, and compaction.
They make the case that most compute time is spent on datacenter/system taxes rather than useful work.
Therefore, since individual function acceleration potential is limited, they propose accelerating big data workloads with a *sea of accelerators* that are chained together in a way that avoids the overhead of core-driven orchestration.
The authors use an analytical model to evaluate accelerator placement, communication and invocation APIs, and potential parallelism of accelerators.

#### Summarize the weaknesses of the paper. length: one paragraph.

Similar weaknesses to the RPC characterization paper - the authors present a bunch of numbers and give some broad 'insights', which aren't even that insightful, and then propose some theoretical accelerators / SoC architecture.
Where do the numbers for the analytical model actually come from? Did the authors closely examine the code paths in the real applications to determine which functions could be offloaded and what each accelerator needs to be capable of? How were the per-accelerator speedups determined? Seems very hand-wavey.
The evaluation of protobuf + SHA3 acceleration has very little to do with the accelerator chaining that's needed for a real application like Spanner.
Accelerator "chaining" isn't that different from accelerators writing to a global memory pool and sharing addresses with each other - is the main issue the orchestration overhead of telling an accelerator when it can fetch results from its data source?
There is no evidence that "chaining" is even viable - there is no codebase dataflow analysis, or kernel extraction, and the case study is completely contrived.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

I would go through the application level traces themselves and identify opportunities for accelerator offloading based on the profiling data - of course, this requires an understanding of what the code is doing to actually determine if offload is viable, and if the algorithm has to be tweaked to make it viable.
What accelerator building blocks are necessary to support a wide variety of application flows? What are the deficiencies of existing CGRA-style architectures that tackle a very similar problem? Are there enough shared compute/filtering/data processing algorithms such that making a set of fixed function accelerators actually makes sense?
