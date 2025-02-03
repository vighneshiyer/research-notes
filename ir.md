# Hardware IR

## Issues We Have Now

So we have FIRRTL. There is a FIRRTL ingestor in circt
HW dialect in circt


In-memory representation
Incrementalism
Dedup natively
Loss of semantics
Unified pass writing mechanism (as rewrite rules ideally)
Native mixed abstraction
Module abstraction is unnecessary and restrictive (port-oriented circuits are a problem for reparenting and other hierarchy manipulation / grouping)

lgraph

Subgraph isomorphism vs graph differences (see LiveSynth, SMatch) (https://en.wikipedia.org/wiki/Subgraph_isomorphism_problem)

Incrementalism
Caching
Both of these are separate problems - consider the SMatch methodology (https://dl.acm.org/doi/pdf/10.1145/3316781.3317912), they focus on the incremental part but across runs there isn't a way to cache common things

The next EDA CAD tool must be able to derive/sample the entire pareto PPA space of designs from the very beginning

egraph based semantic equivalences of subgraphs

at each level of 'lowering' the ability to refer to a group of potential lowered forms is essential. need a way to also 'backtrack' with a common data model as lowering is being performed. try to fix the uarch of behavioral components as late as possible, but also avoid 'fixing' but rather enumerate/sample the space somehow (this is more related to synthesis and PnR, perhaps not to the things we need for an emulator.

Can we leverage an FPGA itself to accelerate graph partitioning, mapping, or subgraph isomorphism?
