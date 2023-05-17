# Parametric Fuzzing / Constrained Random StimGen API

## 5/17/2023

- Rerun spike l1 miss rate fuzzing experiment
    - Also with mark-driven mutation
- Check RTL simulations against spike for commit log consistency and cache behavior

## 4/19/2023

### Paper Ideas

- comparison of input generators
- the value of which feedback metrics to use
    - basic strctural coverage (see TheHuzz for a bunch of these)
        - line, branch, toggle, condition
    - mux-toggle coverage
    - functional coverage
        - related to uArch details - miss rates, pipeline flushes, illegal instructions, memory fetches, ...
    - FSM coverage
    - synthetic coverpoints
        - time-domain coverage of any of the above (hits / cycle in a time window)
        - potentially unreachable conjunctions of any of the above
        - breadcrumbs for temporal properties
        - trajectory coverage (need to either collapse many trajectories in the same category, or only consider trajectories from a particular state for a small number of clock cycles)
    - power model predictions - for power virus generation
- the value of which mutator and selector in the fuzzing loop
- the value of coverage guided formal trace prefix initialization before fuzzing
- the potential to use driller like techniques

## 4/5/2023

- Make sure you have the latest riscv-gcc, compile spike from source, don't use any custom build scripts (since they are usually very old and not maintained anymore)
- Ask someone why spike doesn't terminate as soon as the core write to the tohost memory address
    - I would expect the simulation would terminate immediately
- Let's do some more analysis of the grid search of mark mutation probabilities
    - Just try to look at the common features of the best and worst combinations
    - On just a subset of these, we can do longer runs to prove our hypothesis
- Fuzzing RTL
    - Just printing out the counter on every increment via chisel3.printf
    - There should be no performance issue as long as the stdout goes to a file (simulation runtime should dominate, even with VERBOSE turned on)
- DRAM base: 0x8000_0000
    - RTL sim spends a lot of time in bootrom, only begin counting when we're at the program start
- seems to be a big spike D$ model misses and RTL D$ misses
    - Try to debug by looking at the commit logs from spike and RTL sim in the program execution part (the PCs should match exactly - this is the first step, then we can also look at memory transactions themselves)
    - `spike-dasm` can be run on the commit log from RTL sim to disassemble the instructions
- `riscv64-unknown-elf-objdump -D <elf file>`
    - Look for the write_tohost symbol and the corresponding PC
    - Then, once you reach that PC in RTL simulation, it means you are done with your custom test and you're just sitting in the write_tohost loop
- https://github.com/ucb-bar/riscv-torture
    - "Rewrite" in the parametric generation API
    - Understand their generation strategy (they don't support loops), can we easily port enough features such that we can actually deprecate riscv-torture

## 3/22/2023

- Continue with the grid search across mutator types (biasing towards mutating data vs control bits as determined by their mark)
- Next step is to attempt RTL fuzzing
- Build the RTL simulator in `emulator`: https://github.com/chipsalliance/rocket-chip
    - Use the master branch, not dev
- RocketCore: https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/rocket/RocketCore.scala#LL173C38-L173C45
    - Create a counter (see chisel3.util.Counter) that increments on that event (L1 miss)
    - Use chisel3.dontTouch on it to prevent dead code elimination
- Use a side channel to print out the counter value at the end of simulation
    - https://github.com/chipsalliance/rocket-chip/blob/master/src/main/resources/csrc/emulator.cc#L257
    - `TEST_HARNESS *tile = new TEST_HARNESS;`
    - This `tile` is the C++ Verilated version of the Verilog RTL emitted from running the `rocket-chip` generator
    - Look at the generated `Tile.cc` from Verilator and try to find your L1 miss counter in it by grepping
    - Then, at the end of the main function, print the counter value by accessing that field in Tile
- Alternative to trying to find the counter in Tile.cc
    - It may be easier to create a Verilog blackbox module (using Chisel blackboxes) in RocketChip.scala that uses this annotation (https://verilator.org/guide/latest/connecting.html) (verilator_public) on the counter value passed in as an input
    - Then that signal should be available right from the top-level (`tile->counter`)

## 3/1/2023, Wed

- Spike AFL fuzzing has some weird bug, I will try to reproduce
- Next thing: generator instrumentation for marking pieces of the byte parameters with how they are used
    - Potentially modify the libAFL mutator (or slot in our own) that can use this information
- https://arxiv.org/abs/2209.01789 (ProcessorFuzz paper)
    - Fuzz an instruction stream as usual (completely random), except they focus their mutations on CSR instructions that change the architectural state
    - They only fuzz on ISS! Not RTL, they only test RTL at the end!
    - How does their instruction generator work?

## 2/22/2023, Wed

- Some issues getting AFL to instrument spike, but eventually it does work
- Every (nearly) mutation is ending up corrupting the elf
    - We need a slightly better baseline
    - The mutator should only touch the instruction region of the elf
    - The mutator should directly manipulate the instruction asm stream, and then we use `as` to get an elf
- Use AFL to drive the parametric random generator (AFL wrapper -> Scala -> elf -> spike -> feedback)
- FuzzFactory doesn't make any decisions based on the time-axis (how far along are we in the fuzzing run)
    - We really want to bias which seeds in the corpus we pick for further mutation differently whether we are in the early stages (picking seeds for greater validity) vs the later stages (picking seeds for more basic block length). Idk about AFL here (perhaps they just randomly select seeds from the corpus for mutation, since there is no per-seed info, except maybe a timestamp)
    - We also want to bias how much and what to mutate depending on the time. Idk about what AFL does here...
- https://www.csl.cornell.edu/~cbatten/pdfs/jiang-pyh2-dt2021.pdf
- https://github.com/pymtl/pymtl3/blob/master/examples/ex03_proc/test/inst_bne.py
    - How does their instruction generator work?
    - It looks like their PyH2 code doesn't exist open source, or that the implementation in pymtl is just templated assembly that's filled in with some random instructions. It doesn't look like they do their whole "design the control flow graph" then inject data flow into each node.

## 2/15/2023, Wed

- Perhaps use libAFL instead of AFL
- Easiest path: write a C wrapper around spike that invokes spike as a library with stdin as the program (or use a riscv compiler to generate a good elf)
- TODO: how to use spike as a library, ask Jerry where the sources are, send to Rohit
- start with only basic block coverage, use known good elf files as seeds to AFL
- Big idea: can we bootstrap RTL fuzzing from hardware model fuzzing? Hypothesis: yes, to some degree, but enough to make it worth it.

- Generator instrumentation that tells us what each byte of the parameteric random bytestream is 'marked' as
    - API to 'mark' the usage of a byte as decision, data, intermediate, ...
    - Get out the instrumentation Map + the generated data when calling Gen[T].generate
    - Can't point to the Gen[] object itself that caused the read from the bytestream unless we're OK erasing its type

### Spike as a Library

From Jerry:

> This works (and is regressed on) in spike/master.
> There are no docs for the API, but there is a very simple example here https://github.com/riscv-software-src/riscv-isa-sim/blob/master/ci-tests/test-spike showing how to link against spike and run it.
>
> For single-stepping spike-modeled harts, this https://github.com/ucb-bar/chipyard/blob/main/generators/chipyard/src/main/resources/csrc/cospike.cc is probably the best example.
> Basically you construct an instance of sim_t with the system configuration, and then call sim_t->get_core[hartid]->step() to single-step it.
> The spike source is pretty readable

## 2/8/2023, Wed

1. Comparison of these 3 stimulus generators in an identical fuzzing loop with identical feedback metrics
    - TheHuzz: https://arxiv.org/abs/2201.09941
    - DiFuzzRTL: https://github.com/compsec-snu/difuzz-rtl

2. Can we reuse the infra from JQF to use AFL as a fuzzing engine?

- FuzzFactory: https://github.com/rohanpadhye/FuzzFactory
    - Typical fuzzing feedback: naive feedback metric (basic block coverage) OR custom feedback metric (completed successful parsing / L1 miss rate)
    - Insight: if we combine these feedback metrics, we get much better fuzzing performance (maximization of the custom metric)

3. Can we get basic block coverage out of spike and augment that with our custom metric?
    - Instrument spike with coverage collection of basic blocks
    - Use that as feedback in the fuzzing loop

3b. We can also use gem5 riscv to get CPU model relevant statistics as feedback (or use basic block coverage on gem5, which should be more interesting that spike basic block coverage).

4. Absolute baseline methodology
    - Use AFL to instrument spike
    - spike is fed a completely random sequences of bytes from AFL as a RISC-V elf
        - other option: bytes = RISC-V asm, use `as` to compile it (this avoid the elf construction issues that fuzzing might have)
    - the feedback metric is spike instrumented basic blocks

- Gen debuggability
    - sourcecode: https://github.com/com-lihaoyi/sourcecode

- Vighnesh TODO: write tests for the Gen class itself
    - Hardware datatype library with eager evaluation (to avoid the pain of dealing with signed-only datatypes on the JVM)

## 2/6/2022, Mon

Notes from discussion w/ Kevin:

- Compare generators against other fuzzing papers:
    - DiFuzz generator
    - TheHuzz generator
    - riscv-dv (non-feedback directed)
- Feedback = basic blocks (of spike) + custom metric
- Compare the 3 generators on performance (achieving the maximal custom metric within a time limit)
- Try to reuse AFL as the fuzzing engine
    - Reuse the infra from JQF
    - or could use Graal AOT to compile the generator to a library used with libAFL (https://github.com/AFLplusplus/LibAFL)
- Investigate the 2 other generators (and their feedback metrics and how to isolate generators and feedback)

## 1/25/2022, Wed

- Zest fuzzer works to increase missrate over time
    - ~500 seconds to converge to local maxima, already an excellent result

### TODOs

- Vighnesh's TODOs:
    - Find a suitable arch sim (riscv-perf-model, rivet)
    - Give instructions for compiling a RTL RISC-V simulator for Rocket/BOOM

- Analysis of what stimulus features lead to a higher missrate
- Being able to track which pieces of the random bytestream used as a parameter drive decisions (control) vs data
    - Choosing which sequence to generate = control decision
    - Choosing which register / memory address to r/w from = data decision
    - Choosing an immediate = data decision
- Targeting a different feature
    - Can we use architectural (performance) RISC-V simulators: gem5 RISC-V
    - Metrics: branch mispredict rate, number of pipeline flushes, avg number of entries in ROB, IPC, power, aggregate CPI
- Show that bootstraping a fuzzing run from increasingly refined models up to RTL is much better than fuzzing directly on RTL to begin with + show the power of a functional generative randomization API
    - Target: ICCAD 23 (paper deadline: May 15, 2023)

## 11/9/2022

- Good progress - now we can fix some bugs and move onto generating assembly with the prologue for running on spike
- I can help with that - just send me a snippet of asm
- One note: memory access region is only 0x8000_0000 and up

## 10/26/2022

- https://github.com/riscv/riscv-opcodes

```scala
enum Sequence {
    RAWInstSeq
    ForLoopSeq (body: Gen[Sequence], …)
    AnyInstSeq
    MemSweepSeq
    BasicBlock
    BranchInst (bb1, bb2, cond)
}
```

- Some API to define Sequence to Sequence dependencies (force addresses of two memsweepseqs to be identical, force registers to be reused in subsequent sequence)
- TODO: LLVM IR generators that we can use as a reference?
- spike L2 misses

## 10/19/2022

- Common distributions implemented for biasing

```scala
def keepTrying(Gen[Option[T]]): Gen[T]
```

- How to fix non-tail recursive monadic composition
    - Trampolining (`tailRecM`) https://typelevel.org/cats/typeclasses/monad.html
    - Add ‘keepTrying’ case to your ADT and handle it specially in your interpreter
- RISCV program generation
    - rv32i subset generation works

```scala
val s = Seq(1, 2, 3)
s.pick: Gen[Int] = Gen.oneOf(s)
oneOf(Seq[T]): Gen[T]

val instrs = Map[String, Gen[Instr]] = “add” -> Gen[Instr(“add”, rs1, rs2, rd, …)], Gen[SUB], …
val inst: Gen[Gen[Instr]] = Gen.oneOf(instrs)
val Gen[Seq[Gen[Instr]]] = Seq.fill(1000)(instrs.oneOf()).sequence
```

- Vighnesh:
    - prologue and epilogue of a RISCV asm program (https://github.com/riscv/riscv-test-env/tree/master/p) (https://github.com/riscv-software-src/riscv-tests/tree/master/isa/rv32ui)
    - Instruction set simulator (spike) (https://github.com/riscv-software-src/riscv-isa-sim)
    - Zest like fuzzer for ISS based ‘coverage’ metrics (arch counters)
        - We can start with spike counting cache misses and try to use that as a feedback metric (later we can try using gem5 riscv)
    - Faster generation time vs pyvsc for fixed random knob settings
        - Generative randomization from Scala should achieve better performance vs declarative randomization from Python

- Declarative vs generative random APIs:
    - `Seq[Range]` that is sorted and nonoverlapping

## 10/12/2022

- sequence implementation in terms of ‘traverse’: https://github.com/typelevel/cats/blob/main/core/src/main/scala/cats/Traverse.scala
- https://github.com/TsaiAnson/verif/blob/master/core/src/Randomization.scala
    - https://github.com/ekiwi/maltese
    - Most things in maltese and FIRRTL to TransitionSystem + SMT have been upstreamed to FIRRTL and chiseltest
    - Some code patching is required
- Details on SMT-based constraints
    - The constraint is defined over a Chisel Bundle (see the user-facing API: https://github.com/TsaiAnson/verif/blob/master/core/test/ConstrainedRandomTest.scala)
    - The constraint itself is a function from the Bundle to a chisel Bool (which is just a combinational circuit that emits true when the bundle elements are in a legal state) (see https://github.com/TsaiAnson/verif/blob/master/core/src/Randomization.scala#L16)
    - The constraint (circuit) is compiled to FIRRTL (see https://github.com/TsaiAnson/verif/blob/master/core/src/Randomization.scala#L28) (https://github.com/TsaiAnson/verif/blob/master/core/src/ChiselCompiler.scala)
    - The FIRRTL is converted into a TransitionSystem, this is now a pass in upstream firrtl (https://www.chisel-lang.org/api/firrtl/latest/firrtl/backends/experimental/smt/FirrtlToTransitionSystem$.html)
    - The SMTEmitter pass is run to get an SMT formula
    - The original code uses an SMTSampler algorithm to sample the formula, but you can ignore this for now, instead just use the SMT solver API from chiseltest (https://github.com/ucb-bar/chiseltest/tree/main/src/main/scala/chiseltest/formal/backends/smt) to invoke the solver and get a legal model
- Clone chiseltest
    - Port your random API inside chiseltest - rewriting it in Scala 2 syntax
    - Implement constrained random using what we had earlier in the ‘verif’ repo
    - Think about the API itself
        - Separate the process of generating random stimulus from defining constraints
        - Combine constraints that are written separately
        - Turn on/off constraints ‘dynamically’ (look at the SystemVerilog constrained random API for how they do things, why that’s bad a idea - can we write a better API)
    - Show that our library is competitive with SOTA
        - PyVSC
            - Go through some of their examples
            - Write them using generative style
            - Write them using declarative style
            - See the performance differences between PyVSC (declarative only) vs. our thing which supports both styles
        - https://github.com/chiselverify/chiselverify
            - Take a look at their API
            - Under the hood it uses a CSP solver and native Scala classes as randomization targets
        - Generative: rand N in 10:20, generate each Rand Int, Seq[N]
        - Declarative: rand [10:20][Int]
    - Biases and distributions and how to override them nicely
        - Look at ScalaCheck for APi inspiration
    - Functional coverage of stimulus

## 10/5/2022

1. Constrained random API (QuickCheck-like, but with additional features like declarative constraints)
    a. Gen[Seq[DecoupledTransaction]]] is a description of how to generate a random sequence of decoupled transactions
    b. An interpreter can inject either *true* randomness or parametric (i.e. fake, fully deterministic) randomness to the generated object
        i. Among many other features of the interpreter (e.g. introspection, instrumentation of decisions, etc.)

- Rohit: Write a purely generative calculator expression generator
    - Draft the API from bottom up - see the imperative generator in Zest (https://dl.acm.org/doi/pdf/10.1145/3293882.3330576)
    - Once we have that working, we can think about the atoms we need in the functional random API
    - Here’s a draft:

```scala
val numExprs: Gen[Int] = Gen[Int].range(1, 10)
enum Op {PLUS, MINUS, ...}
val opGen = Gen.oneOf(Op, bias = Uniform)

def genExpr(op: Gen[Op], literals: Gen[(Int, Int)]): Gen[Expr] = {
    op <- op
    lit1, lit2 <- literals
} yield Expr(op, lit1, lit2)

val exprs: Gen[Seq[Expr]] = numExprs.flatMap { n => Seq.fill(n)(genExpr()) }.sequence

exprs.randomize(useScalaRandom, useFixedRandom(Stream[Byte])) -> Seq[Expr]
```

- Start off very simple, just concretize the type of the random monad e.g. Gen[Int] and write a generator for it gen(Gen[Int]): Int
- Then make it type generic
- See someone’s implementation of Ch. 8 of FP in Scala (https://github.com/bmatsuo/functional-programming-in-scala/blob/master/src/main/scala/com/fp/ch6/RNG.scala)
- Here is a real library example (https://github.com/NICTA/rng)
    - See how `nextbits` is used here (https://github.com/NICTA/rng/blob/master/src/main/scala/com/nicta/rng/Rng.scala)
- More explanation here: https://typelevel.org/cats/datatypes/state.html
