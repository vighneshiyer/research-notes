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
Cross-module optimizations and DCE without flattening
Uniform treatment of constant propagation vs ports (e.g. consider the hart id input into a RocketTile)
SSA form itself is a problem
'views' -> having a view of a restricted IR and no hierarchy from the perspective of a pass, but there is translation layer that applies the modifications to the hierarchy

lgraph

Subgraph isomorphism vs graph differences (see LiveSynth, SMatch) (https://en.wikipedia.org/wiki/Subgraph_isomorphism_problem)

Incrementalism
Caching
Both of these are separate problems - consider the SMatch methodology (https://dl.acm.org/doi/pdf/10.1145/3316781.3317912), they focus on the incremental part but across runs there isn't a way to cache common things

The next EDA CAD tool must be able to derive/sample the entire pareto PPA space of designs from the very beginning

egraph based semantic equivalences of subgraphs

at each level of 'lowering' the ability to refer to a group of potential lowered forms is essential. need a way to also 'backtrack' with a common data model as lowering is being performed. try to fix the uarch of behavioral components as late as possible, but also avoid 'fixing' but rather enumerate/sample the space somehow (this is more related to synthesis and PnR, perhaps not to the things we need for an emulator.

Can we leverage an FPGA itself to accelerate graph partitioning, mapping, or subgraph isomorphism?

- would be useful to look at how cranelift is different from LLVM: https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/docs/compare-llvm.md
  - cranelift uses a single IR whereas LLVM uses multiple IRs at different levels of the compiler pipeline. cranelift IR is not as amenable to mid-level opt.
    - it seems that cranelift's IR nodes have empty fields that are filled in by passes rather than being transformed into new node types with new fields

- https://www.cs.cornell.edu/~asampson/blog/flattening.html
- https://www.cs.cornell.edu/~asampson/blog/flatgfa.html

- Some way to store diffs of graphs in parallel with the hashed graph -> AST structure
 - So if you're doing some cross-module optimization (e.g. constant propagation, DCE), then ideally you don't want to flatten everything (inline every module) and treat all the instances of the same module uniquely (this is how things are done today in every CAD tool, even in Verilator, and others)
 - Instead, you want to store some 'base' module description, and then store diffs of that base in every place that the module is instantiated but optimized differently
 - This is somewhat a duplicate

- Thinking on the synthesis and PnR side. what we really want is to have a space of options (of microarchitecture, and PPA tradeoffs) for any given subgraph and then this becomes an ideal composition problem (similar to the extraction problem for egraphs).
  - This meshes with the 'blob-oriented' physically-aware synthesis concept

- A taxonomy of hardware compiler passes
  - Write/read (modification) vs read-only (analysis)
  - Modifications unique per module instance (context-aware) vs independent of module instance (uniform)
  - Parallelizable with respect to subcircuits (is topological sorting required, or can trivial graph parallel algorithms work)
  - Can a pass affect the module hierarchy? By reparenting, splitting, or otherwise. Reparenting can be tricky since you might want to grab an instance and various bits of logic attached to it, and deal with them uniformly.
- Passes: CSE, DCE, constant propagation, FAME0, FAME1, FAME5, FAME memory optimizations, assertion/cover synthesis, width inference, type resolution (input/output), SSA to graph transformations, statement-oriented transformations (e.g. case/if to mux conversion), print synthesis, comb loop detection, reset type conversion, register init, memory transformations / mappings, module deduplication, module renaming, port punching / pushing internal signals out to the top level, module grouping, module inlining
