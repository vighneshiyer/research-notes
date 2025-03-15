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
 - This is somewhat a duplicate form of the hash based AST database; unclear what value having a separate diff database would get you. Furthermore, representing graph diffs might be tricky.

- Thinking on the synthesis and PnR side. what we really want is to have a space of options (of microarchitecture, and PPA tradeoffs) for any given subgraph and then this becomes an ideal composition problem (similar to the extraction problem for egraphs).
  - This meshes with the 'blob-oriented' physically-aware synthesis concept

- A taxonomy of hardware compiler passes
  - Write/read (modification) vs read-only (analysis)
  - Modifications unique per module instance (context-aware) vs independent of module instance (uniform)
  - Parallelizable with respect to subcircuits (is topological sorting required, or can trivial graph parallel algorithms work)
  - Can a pass affect the module hierarchy? By reparenting, splitting, or otherwise. Reparenting can be tricky since you might want to grab an instance and various bits of logic attached to it, and deal with them uniformly.
- Passes: CSE, DCE, constant propagation, FAME0, FAME1, FAME5, FAME memory optimizations, assertion/cover synthesis, width inference, type resolution (input/output), SSA to graph transformations, statement-oriented transformations (e.g. case/if to mux conversion), print synthesis, comb loop detection, reset type conversion, register init, memory transformations / mappings, module deduplication, module renaming, port punching / pushing internal signals out to the top level, module grouping, module inlining

Structural vs behavioral constructs in hardware and software

- event driven semantics and all derived state
- behavioral semantics with specified state
  - behavioral control flow with structural dataflow
- structural semantics

- what is behavioral or structural in the first place?
- behavioral: if you're describing the behavior of some construct that can be expressed in multiple ways structurally
- structurally: is defined *with respect to* some set of primitives - this is different based on the target you're talking about

- Capturing subgraph functions with holes that can be filled in as part of the caching layer
  - How abstract should 'equivalent' representations be? e.g. if a circuit has the same structure besides bitwidth, should those be treated as the same? what about entries in an array or so on... how much can we abstract away and substitute as inputs?
- abstraction vs paradigm vs structural/behavioral, are these all saying the same thing?
- how do you express a clock mux? or any other type of structural thing that might be behaviorally described just as well.

- all hardware IRs today quickly move to structural representation
  - FIRRTL, LLHD, Circt's representation for other dialects, LNAST/LGraph/LiveHD
  - We think we shouldn't do this and instead preserve a super-phi-node sea-of-nodes representation straight from the behavioral representation
- My chat (updated): https://chatgpt.com/share/67c8de3e-01a4-8004-9f29-822163958373

- Dan's articles
  - Event-Driven & Reactive Hardware: https://archive.is/ouPsl
  - Models All The Way Down: https://archive.is/PRFHI#selection-302.0-302.1
  - The Languages of Hardware: https://archive.is/xNVYt#selection-302.0-302.1
  - How Software Makes Hardware: https://archive.is/MxFVO#selection-302.0-302.1

- So there are clear downsides to the all behavioral approach to hardware IR control flow, but I feel those are all OK
  - Joonho will ask Jack about this
  - One major downside is there is no way to trace the driver tree of a given net exactly and fully localize its control expression
  - Comb loop detection becomes harder too
  - You can't do cell type counting (e.g. count the muxes), you have to
  - How does this impact bitwidth inference and deduplication?

- Deduplication, caching, incrementalism, and abstraction are all sides of the same coin

- A naturally deduplicating hardware representation would ideally do away with the notion of 'modules' as an IR primitive, rather making modules into pure metadata around an arbitrary blob of logic that fundamentally marks replicated logic
  - The deduplication/caching/incremental technique should be independent of the graph schema itself, and the semantics of each graph node + edge attribute
  - Consider the fully structural case first

## Random Notes

- Consider a content-addressed IR
  - `val x = 1.U + 2.U`
  - This uses a primitive `+: UInt => UInt`, hashes to something (let's say all primops on a given HW type have some hash e.g. `H+`)
  - `[nameless] = (() -> H+ UIntLit(1) UIntLit(3)` (no input args, literals are explicit value types and not mere bindings, perhaps they have their own hashed representation)
  - `UInt.+ = (#arg1 #arg2 -> #H+ arg1 arg2`. Definitions on type `UInt`. Is this signature any different from `H+` itself? Perhaps that is all `H+` is!
- Questions for the frontend
  - Should every expression return a concrete type?
  - Should the frontend aggressively rewrite sub-expressions with their hashes, or leave some duplication?
  - Should type constructors (e.g. `T -> Vec[T]`) be a part of the IR?
  - Can we sketch out how parameters can unify plusargs and inputs (e.g. `hart_id` fiasco in Rocket)
  - Unified optimization strategy (equivalent wrt semantics NOT structure) - how can we handle optimized out parts of a circuit that are context-dependent?
