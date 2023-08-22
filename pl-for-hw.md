# PL for HW Reading Group

- An overview of type system formalism
    - https://langdev.stackexchange.com/questions/2692/how-should-i-read-type-system-notation
    - https://stackoverflow.com/questions/12532552/what-part-of-hindley-milner-do-you-not-understand

## 2/21/2023 - PyMTL3, PyH2, SystemC TLM

### PyMTL3

- Claims modeling of RTL, cycle-level (CL) (timed, but manually annotated), and functional-level (FL) (untimed) models
- Claims multilevel modeling - swap out models at transaction port boundaries, simulation kernel is in Python
- They have their own IR (MIR) with a stable user-targettable Python API for programmatic manipulation / instrumentation
    - They can ingest FIRRTL and turn it into MIR
    - MIR supports CL/FL models with function invocation to/from RTL models (functions may be arbitrary Python - not serializable by default)
- Their RTL representation is *event*-based (similar to behavioral LLHD / Verilog) not *primitive*-based (like FIRRTL)
- Unified modular ordering constraints (UMOC) - this paper seemed dumb when I first read it, but it may have more meaning when considered outside the RTL context
    - At RTL-level, this just boils down to combinational loop detection
    - When mixing models, it might be some kind of fixpoint iteration
- PyMTL3 supports a Verilator simulation backend - **this is a target for benchmarking SimCommand against** (is it similar to cocotb, or superior)?
    - They make an **odd claim** about cocotb - saying that because it "embeds Python in Verilog", not all Python constructs can be used (this seems wrong)
- Leverage Python ecosystem
    - pip for packaging
    - pytest for unit testing
    - hypothesis for property-based testing (covered in PyH2 paper)
- They claim that dynamic typing is an *advantage*
    - Crazy API allows you to modify parameters of an RTL block using string specifiers after they have been instantiated, but before elaboration
    - They mention the problem of having to pass down the entire SoC config to every leaf component - they can just use hierarchical reference parameters from the top
        - This is unnecessary, first of all - you can just pass the relevant fragment of the config
        - Rocket-chip does this in a very ugly way with the Parameters object
        - But more importantly, having a top-level config is a *good thing* because it forces a single place that controls parameterization
            - In their world, any module can have its parameters mutated by any other
- Their CheckClockGatingPass seems useful as a first pass to make sure all registers have *some* gating logic (even if unused)
    - The FIRRTL lowering could break this though. e.g. a counter with an en signal has a well gated counter reg, but if at a higher-level that en is tied to hard 1, then the if statement will be constprop-ed out and counter will no longer be well gated (but that's what you want in that case)
- AreaEstimationPass is also useful, VerilogTBGenPass - useful for GL simulations, generates plain vlog testbenches
- Their Mamba PyPy based simulator is interesting **explore more**
- They seem to preserve interface/structs from Python to SystemVerilog emission, their emission framework also seems very usable (easy to modify)
- UMOC seems to provide some ability to invoke multiple transactions on multiple ports on the same timestep between functional and RTL sim - this only seems to work when a explicit ordering decision is made in dynamic simulation, turning it into a formal model is probably tough

- Hypothesis: PyMTL3 realizes the agile hardware manifesto.
    - Does this hold?
    - Can it hold its own against SystemC and SV methodologies? (consider simulation speed, expressivity, available abstractions, dependencies, test frameworks)
- Method vs port/transaction modeling
    - Method: order of calls must be serialized by the caller, can mutate arbitrary state, can have a 'return' value, arguments are distinct to each function, less similar to final hardware implementation, not event based (manual triggers required), when is it legal to call a method?
        - Optional mapping from method call to underlying RTL port interaction (can be standardized as in the case of PyMTL, or general like in paso)
        - Potential for mismatch between method calls and overlapping RTL port driving (which may not be feasible in RTL, but may appear to be in FL/CL models)
    - Port/transaction: model ports map to hardware ports, transactions also map, transaction types distinct to each port, parallel port interaction is 'natively' understood, interaction uses fork/join vs serial method calls assumed to take place on the same cycle if possible, always legal to attempt to drive a transaction but the driver may gate it on a given cycle, maps to typical UVM-style DV environment, transactions are naturally event based and can trigger (multiple, parallel) things inside the module
    - Ultimately these both can be reduced to one abstraction

### PyH2

- PBT:
    - doesn't generate a full random stimuli ahead of time, it is done lazily as the simulation proceeds, possibility to use simulation feedback to affect generation
- **Idea**: unify the work of parametric generators and PBT autoshrinking - this already kind of exists for QuickCheck, but we want to discover hardware specific algorithms
- They see rfuzz (mutation cov-guided fuzzing) as distinct from PBT
    - **But these are really points on a continuum**
- Using coverage.py with pytest seems nice - easy to instrument testbench and RTL code without any separate compilation steps
- Using pytest's test parameterization is also slick - it is better than sweeping parameters within a test function since each function run is treated independently (and can be parallelized) and the test cases are automatically named
- Figure 1b) is misleading, or perhaps too general
- 1c with our API:

```scala
val gen = Gen[Int].range(1, 128)
val tx = for {
    a <- gen
    b <- gen
} yield (a, b)
val txns: Gen[Seq[(Int, Int)]] = Seq.fill(100)(tx).sequence
// full random
val crt: Gen[Seq[Bool]] = txns.flatMap{ t => t.map{ case a, b => gcd(a, b) == math.gcd(a, b) } }
crt.generate(): Seq[Bool]
// hypothesis
val s: LazyList[(Int, Int)] = tx.generateStream()
// invoke quickcheck runtime (need some side channel to affect randomization by controlling the parametric bytestream)
```

- **Look into PBT libraries and the strategies they use to manipulate their parametric bytestreams** (some of this can carry over into our parametric fuzzing experiments)
- **Look into the RuleBasedStateMachine** class in hypothesis - seems to be a nice abstraction for writing properties over stateful systems
- PBT is really just CRT with lazy stimulus generation and autoshrinking, they are the same dynamic testing technique under the hood
- Smart technique to use Gen datatypes for the design parameters + stimulus - this means they can piggyback on existing shrinking techniques in hypothesis (with a unified parametric stream that they use for each type of randomization)
- **Look at PyH2P**'s asm program template approach, generate control flow template, then fill in basic blocks

### SystemC TLM

- SystemC ports, carry TLM transport interfaces
    - Interface types: bidirectional blocking, unidirectional blocking/nb
    - Interface invoked with `transport()` method inside a SC_METHOD or SC_THREAD
- Port vs interface vs method vs channel
- wait() in a SC_THREAD is a context switch? Modeling parallelism via threading would seem to have quite high overhead.
- A lot of complication, I'm still unsure what the basic primitives are and how the simulation kernel schedules their interactions
