# Parametric Fuzzing / Constrained Random StimGen API

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

• Good progress - now we can fix some bugs and move onto generating assembly with the prologue for running on spike
• I can help with that - just send me a snippet of asm
• One note: memory access region is only 0x8000_0000 and up

## 10/26/2022

• https://github.com/riscv/riscv-opcodes
• enum Sequence
    ◦ RAWInstSeq
    ◦ ForLoopSeq (body: Gen[Sequence], …)
    ◦ AnyInstSeq
    ◦ MemSweepSeq
    ◦ BasicBlock
    ◦ BranchInst (bb1, bb2, cond)
• Some API to define Sequence to Sequence dependencies (force addresses of two memsweepseqs to be identical, force registers to be reused in subsequent sequence)
• TODO: LLVM IR generators that we can use as a reference?
• spike L2 misses

## 10/19/2022

Rohit:
    • Common distributions implemented for biasing
def keepTrying(Gen[Option[T]]): Gen[T]
    • How to fix non-tail recursive monadic composition
        ◦ Trampolining (tailRecM) https://typelevel.org/cats/typeclasses/monad.html
        ◦ Add ‘keepTrying’ case to your ADT and handle it specially in your interpreter
    • RISCV program generation
        ◦ rv32i subset generation works
val s = Seq(1, 2, 3)
s.pick: Gen[Int] = Gen.oneOf(s)
oneOf(Seq[T]): Gen[T]

val instrs = Map[String, Gen[Instr]] = “add” -> Gen[Instr(“add”, rs1, rs2, rd, …)], Gen[SUB], …
val inst: Gen[Gen[Instr]] = Gen.oneOf(instrs)
val Gen[Seq[Gen[Instr]]] = Seq.fill(1000)(instrs.oneOf()).sequence

• Vighnesh:
    ◦ prologue and epilogue of a RISCV asm program (https://github.com/riscv/riscv-test-env/tree/master/p) (https://github.com/riscv-software-src/riscv-tests/tree/master/isa/rv32ui)
    ◦ Instruction set simulator (spike) (https://github.com/riscv-software-src/riscv-isa-sim)
    ◦ Zest like fuzzer for ISS based ‘coverage’ metrics (arch counters)
        ▪ We can start with spike counting cache misses and try to use that as a feedback metric (later we can try using gem5 riscv)
    ◦ Faster generation time vs pyvsc for fixed random knob settings
        ▪ Generative randomization from Scala should achieve better performance vs declarative randomization from Python

Declarative vs generative random APIs:
    Seq[Range] that is sorted and nonoverlapping

## 10/12/2022

Rohit:
    • sequence implementation in terms of ‘traverse’: https://github.com/typelevel/cats/blob/main/core/src/main/scala/cats/Traverse.scala
    • https://github.com/TsaiAnson/verif/blob/master/core/src/Randomization.scala
        ◦ https://github.com/ekiwi/maltese
        ◦ Most things in maltese and FIRRTL to TransitionSystem + SMT have been upstreamed to FIRRTL and chiseltest
        ◦ Some code patching is required
    • Details on SMT-based constraints
        ◦ The constraint is defined over a Chisel Bundle (see the user-facing API: https://github.com/TsaiAnson/verif/blob/master/core/test/ConstrainedRandomTest.scala)
        ◦ The constraint itself is a function from the Bundle to a chisel Bool (which is just a combinational circuit that emits true when the bundle elements are in a legal state) (see https://github.com/TsaiAnson/verif/blob/master/core/src/Randomization.scala#L16)
        ◦ The constraint (circuit) is compiled to FIRRTL (see https://github.com/TsaiAnson/verif/blob/master/core/src/Randomization.scala#L28) (https://github.com/TsaiAnson/verif/blob/master/core/src/ChiselCompiler.scala)
        ◦ The FIRRTL is converted into a TransitionSystem, this is now a pass in upstream firrtl (https://www.chisel-lang.org/api/firrtl/latest/firrtl/backends/experimental/smt/FirrtlToTransitionSystem$.html)
        ◦ The SMTEmitter pass is run to get an SMT formula
        ◦ The original code uses an SMTSampler algorithm to sample the formula, but you can ignore this for now, instead just use the SMT solver API from chiseltest (https://github.com/ucb-bar/chiseltest/tree/main/src/main/scala/chiseltest/formal/backends/smt) to invoke the solver and get a legal model
    • Clone chiseltest
        ◦ Port your random API inside chiseltest - rewriting it in Scala 2 syntax
        ◦ Implement constrained random using what we had earlier in the ‘verif’ repo
        ◦ Think about the API itself
            ▪ Separate the process of generating random stimulus from defining constraints
            ▪ Combine constraints that are written separately
            ▪ Turn on/off constraints ‘dynamically’ (look at the SystemVerilog constrained random API for how they do things, why that’s bad a idea - can we write a better API)
        ◦ Show that our library is competitive with SOTA
            ▪ PyVSC
                • Go through some of their examples
                • Write them using generative style
                • Write them using declarative style
                • See the performance differences between PyVSC (declarative only) vs. our thing which supports both styles
            ▪ https://github.com/chiselverify/chiselverify
                • Take a look at their API
                • Under the hood it uses a CSP solver and native Scala classes as randomization targets
            ▪ Generative: rand N in 10:20, generate each Rand Int, Seq[N]
            ▪ Declarative: rand [10:20][Int]
        ◦ Biases and distributions and how to override them nicely
            ▪ Look at ScalaCheck for APi inspiration
        ◦ Functional coverage of stimulus

## 10/5/2022

1. Constrained random API (QuickCheck-like, but with additional features like declarative constraints)
    a. Gen[Seq[DecoupledTransaction]]] is a description of how to generate a random sequence of decoupled transactions
    b. An interpreter can inject either *true* randomness or parametric (i.e. fake, fully deterministic) randomness to the generated object
        i. Among many other features of the interpreter (e.g. introspection, instrumentation of decisions, etc.)

• Rohit: Write a purely generative calculator expression generator
    ◦ Draft the API from bottom up - see the imperative generator in Zest (https://dl.acm.org/doi/pdf/10.1145/3293882.3330576)
    ◦ Once we have that working, we can think about the atoms we need in the functional random API
    ◦ Here’s a draft:

val numExprs: Gen[Int] = Gen[Int].range(1, 10)
enum Op {PLUS, MINUS, ...}
val opGen = Gen.oneOf(Op, bias = Uniform)

def genExpr(op: Gen[Op], literals: Gen[(Int, Int]]): Gen[Expr] = {
for {
op <- op
(lit1, lit2) <- literals
} yield Expr(op, lit1, lit2) }

val exprs: Gen[Seq[Expr]] = numExprs.flatMap { n => Seq.fill(n)(genExpr()) }.sequence

exprs.randomize(useScalaRandom, useFixedRandom(Stream[Byte])) -> Seq[Expr]

◦ Start off very simple, just concretize the type of the random monad e.g. Gen[Int] and write a generator for it gen(Gen[Int]): Int
◦ Then make it type generic
◦ See someone’s implementation of Ch. 8 of FP in Scala (https://github.com/bmatsuo/functional-programming-in-scala/blob/master/src/main/scala/com/fp/ch6/RNG.scala)
◦ Here is a real library example (https://github.com/NICTA/rng)
    ▪ See how nextbits is used here (https://github.com/NICTA/rng/blob/master/src/main/scala/com/nicta/rng/Rng.scala)
◦ More explanation here: https://typelevel.org/cats/datatypes/state.html
