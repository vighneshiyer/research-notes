# Chisel Sequences

## 2/1/2023

- Retreat is May 22-24 (Mon-Wed), 2023
- ScalaSeq -> PSL formula
    - WIP: PSL formula -> HOA (by calling SPOT) (use sequences.backend.Spot.callSpot)
    - WIP: interpret the HOA over a concrete Scala trace (and compare it to our manual implementation of assert/cover of Scala sequences)
- Testing more sequences types for the SpotBackend (likely many of them aren't being handled properly)
    - There is a bug! PSL concat of 3 or more APs doesn't yield the correct FSM.
    - Investigate the correct encoding of multiple AP concats
    - Incorrect way: a & (X b & (X c))
    - Correct way (but ugly): a & (X b) & (X X c)
    - Click help here (https://spot.lre.epita.fr/app/) and look at the section "PSL Infix Syntax"
    - Investigate the concatenation and fusion operators
- Another Spot test line 71 - fails for no good reason
    - Suspect that the PSL emission of `SeqConcat(a, SeqOr(b, notC))` is wrong
    - Try unit testing this, and manually checking the FSM using Spot's webapp
- Just do something like this to get around the sequence wrapped in a Property
```scala
case SeqImplies(s1, p1) =>
val s = p1 match {
  PropSeq(s) => s
}
s"(${sequenceToPSL(s1)} |->${s})"
```

## 1/25/2023

- Send you details about next retreat (late May)
- tapeout, 140, 130/189, 152, EE 219C (formal methods)

### Vighnesh's TODOs

- Rebase on top of Kevin's upstream
- Create a dev branch on my repo (and you can merge your stuff on that dev branch)
- Unify the testing infrastructure for SPOT and SequenceFSM backends

### TODOs

- Check on the Scala sequence to SPOT convertion + HOA parsing + assertions

- Write sequences for Saturn L1 cache identical to the ones currently written in SystemVerilog
- `SpotBackendTests.scala` write some tests that exercise all the potential PSL properties that we can emit from Chisel
    - Goal: verify that the automata generated from the PSL -> SPOT -> HOA -> Chisel circuit MATCHES the interpretation of the raw HOA automata in Scala
    - Tasks:
        - Continue work on scala sequences - to convert ScalaSeq -> Spot PSL string -> HOA
        - Validate our HOA processing engine `def process(h: HOA, apTrace: Seq[Set(Int)]: (Bool, Option[Int])` (golden model for what the Chisel circuit should do)
        - Extend tests in `SpotBackendTests.scala` to cover more types of properties
            - So far, only `SeqConcat`
            - Can we also test `SeqFuse`, `SeqOr`, `SeqAnd`, multiple chained `SeqConcat`?
    - Eventual flow:
        - Generate a bunch of properties in the Sequence ADT
        - From each property: generate its HOA automata and Chisel monitor circuit
        - Fuzz both the automata and the Chisel monitor circuit to check for equivalence
    - Later:
        - Formally prove equivalence between a SPOT HOA and a Chisel circuit representation

## 12/6/2022

• TODO Vighnesh: rewrite the HOA parser so that everything is mapped on the original String values passed into SPOT as atomic propositions (rather than arbitrary integers that are assigned in the HOA header to each AP).
• TODO Vighnesh: rewrite the sequences frontend so that it can work universally with Chisel and Scala based sequences
    ◦ Can we also unify the ADTs themselves?

```scala
sequence[Queue, Any] {
    val wr_ptr_max = AP(wr_ptr == queue.params.depth)
}
```

• TODO Vighnesh: gather a bunch of examples of temporal properties that we can use to benchmark the different implementations we have
    ◦ Pure software impl - interpretation (with FFI)
    ◦ Pure software impl - Java code generation and JIT compilation (with FFI)
    ◦ SPOT -> HOA in software (with FFI)
    ◦ Software impl -> convert to C++ -> compile with RTL simulator
    ◦ Prop -> Chisel RTL evaluation in RTL simulation
• TODO: implement constructFormula
• TODO: use the map returned from constructFormula inside assertHOA to actually evaluate the HOA automata for a given trace
• Next step: figure out how to reintroduce local state for SPOT based automata checkers

## 11/16/2022

• Some fixes required for the assert tests (remember that asserts are checked globally just like covers so if there is a condition in the start of the assert it is checked on every element of the trace - there can be failures on the same timestep)
• TODO: try to consolidate the assert and cover functions (keep them separate, but the underlying functions they call should be very similar and just have flags that change their behavior slightly)
• TODO: state support
    ◦ DONE! live
• Things to do:
    ◦ TileLink / AXI4 monitor (https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tilelink/Bundles.scala)
    ◦ SPOT integration
        ▪ Look at `spot.scala`
        ▪ `val psl = s"G(${sequenceToPSL(seq)})"`
        ▪ `val hoaString = callSpot(psl)`
        ▪ `val hoa = HOAParser.parseHOA(hoaString)`
        ▪ ScalaSeq -> PSL/LTL string
        ▪ Call spot
        ▪ Get back the HOA automata
        ▪ Write assert in terms of the HOA automata
            • assert(trace: T, hoa: HOA): AssertResult
        ▪ ltl2tgba -B -D -f "G (a -> X b)"
        ▪ Ignore state for now

## 11/9/2022

• Implementation of Or and Implies in the interpreter
• Prototyping per-property and global across-property state
• TODO Vighnesh: give push permission to Andy on chisel-sequences
• TODO: add assert function
• TODO: fix bugs in Or and add additional unit tests for those
• TODO: state implementation

## 11/2/2022

• Look through the implementation of Scala level sequences and the cover checker here (https://github.com/ekiwi/chisel-sequences/blob/main/test/scala_sequences/SequenceCoverSpec.scala)
• Be able to run all the tests
• Add some more tests, notice that there are several nonimplemented primitives in my implementation (e.g. Or, …) (look for “???”)
    ◦ Implement Or and Implies and write tests for them
• Make sure you can use IntelliJ as an IDE (or vscode)
• TODO: prototype local state support for sequences (per-instance state for a given sequence)

```scala
sealed trait ScalaSeq[+T, +S]
case class AtmProp[T, S](pred: (T, S) => Boolean, update: (S, T) => S) extends ScalaSeq[T, S]
case class Fuse[T, S](seq1: ScalaSeq[T, S], seq2: ScalaSeq[T, S]) extends ScalaSeq[T, S]
```

## 10/26/2022

• https://github.com/ekiwi/chisel-sequences
    ◦ Spot backend for Chisel sequences works (can generate a property checker by parsing the automata returned by Spot and turning it into a Chisel Module)
    ◦ Goal is to integrate the Scala sequences ADT in this repo and be able to use Spot as a backend too
• Given transactions on the A and D channels of a TL interface, report any violations by checking properties on them
    ◦ https://starfivetech.com/uploads/tilelink_spec_1.8.1.pdf
    ◦ Goal eventually is rocket-chip TLMonitor replacement (https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tilelink/Monitor.scala)
    ◦ https://github.com/TsaiAnson/verif/blob/master/tilelink/test/TLSLPropertyTest.scala
• OpenTitan properties: https://github.com/lowRISC/opentitan/blob/55bb57493e93c2a4508dbd20f1e75c67c55cd7e9/hw/ip/tlul/rtl/tlul_assert.sv#L319
• FIFO RTL test, enq/deq
    ◦ val enqTxns = Seq[Transaction(data, timestamp) | Nop]
    ◦ val deqTxns = Seq[Transaction(data, timestamp) | Nop]

```systemverilog
sequence enqFire enq.ready && enq.valid
sequence deqFire(d) deq.ready && deq.valid && deq.data == d
assert enqFire {d = enq.data} ##[1:*] deqFire(d)
```

https://www.doulos.com/knowhow/systemverilog/sva-properties-for-pipelined-protocols/
SVA local variables

```systemverilog
property fifo_is_consistent;
  logic [31:0] d;
  ( enq.valid && enq.ready, d = enq.data ) |=>
   ##[1:*] (deq.valid && deq.ready && (deq.data == d));
endproperty
```

```scala
sealed trait Sequence[+T, +S]
case class AtomicProp[T, S] (fn: (T, S) => Boolean, stateUpdate: Optional[S => S] = None) extends Sequence[T]
```

## 10/19/2022

Andy:
    • Vighnesh: Pure Scala implementation of sequences over generic type T and a checker for those (without using Spot)
    • TODO: continue working on the check function, and reference my code (TBD)
    • Goal eventually is rocket-chip TLMonitor replacement (https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tilelink/Monitor.scala)
        ◦ AXI protocol: https://developer.arm.com/documentation/ihi0022/e/AMBA-AXI3-and-AXI4-Protocol-Specification/Introduction/About-the-AXI-protocol?lang=en
    • First open source sequence / temporal prop spec API in existence

## 10/12/2022

Andy:
    • Sequence tracking implementations
        ◦ Manual tracking of state, duplicate sequence state every time the sequence is ‘started’, and track on a index-by-index basis as you traverse the list
        ◦ Something more systematic: take your sequence description, convert it to a standard specification language (PSL, LTL), give that to a automata tool (e.g. spot), spot will give you an automaton, then you interpret that automaton in Scala
    • https://spot.lrde.epita.fr/
        ◦ https://spot.lrde.epita.fr/concepts.html (READ THIS)
        ◦ PLAY here: https://spot.lrde.epita.fr/app/
    • SVA
        ◦ https://www.doulos.com/knowhow/systemverilog/systemverilog-tutorials/systemverilog-assertions-tutorial/

## 10/5/2022

3. Temporal property specification language
    a. LTL (G a -> X b)
    b. Atoms are: atomic propositions and operators on those APs
    c. SVA (SystemVerilog assertions)

• Andy: instead of jumping straight to properties, let’s just check where sequences are present on a Seq[T]
    ◦ https://github.com/ekiwi/chisel-sequences (we will get our code in here to merge with the Chisel sequences implementation - we want to unify the API for chisel and pure Scala sequences)

val isTrue = atomicProp[Bool](i => i)
val isFalse = atomicProp[Bool](i => !i)
val trueThenFalse = Concat(isTrue, isFalse) // one “cycle” between true and false
ConcatFuse(isTrue, isFalse) // both are true on the same cycle (never possible in this case, but useful for generic sequences over some T)

def check[T](seq: Sequence[T], trace: Seq[T]): (completed: Seq[(Int, Int)], pending: Seq[Int])

check(trueThenFalse, Seq(true, false, true, false, false, true))
== ([(0, 1), (2, 3)], [5])

• Play with more complex sequences on different types T and write unit tests for check
• Next week, we can move on to property definitions and unification with chisel sequences

## 9/30/2022

• Repo: https://github.com/ekiwi/chisel-sequences
• SVA vs PSL: PSL AND SVA: TWO STANDARD ASSERTION LANGUAGES ADDRESSING COMPLEMENTARY ENGINEERING NEEDS
• Unification of property APIs across testbenches and RTL
    ◦ Same API can describe properties across Chisel Data and for Streams/Seqs of any Scala datatype
    ◦ Ideally, a property should be testable outside a chiseltest/Chisel3 context
• Interpretation variants:
    ◦ Direct SVA string emission in a Verilog blackbox
    ◦ Testbench-connected software (Scala) interpretation / streaming Scala interpreter
    ◦ RTL monitor emission via BlueSpec-like reduction
    ◦ RTL monitor for formal (property start point becomes a free variable for the solver to choose)
    ◦ Scala -> Native (machine code monitor) for linking within Verilator/VCS simulation
• Alternative backends (vs BSV technique)
    ◦ SPOT (https://spot.lrde.epita.fr/)
• Other attempts of creating a property specification language
    ◦ GHDL PSL
    ◦ Yosys SVA to FSM
        ▪ https://tomverbeure.github.io/rtl/2019/01/04/Under-the-Hood-of-Formal-Verification.html
        ▪ Generating Hardware Assertion Checkers - Marc Boule
    ◦ https://github.com/iscas-tis/CHA (Chisel SVA attempt)
    ◦ Investigate if anything exists for SpinalHDL
• TODOs:
    ◦ User-facing API to construct sequences/properties
    ◦ Can we describe recursive properties / sequences?
    ◦ Better testing (unified API for interpreted, synthesized, code gen’ed, etc. properties)
