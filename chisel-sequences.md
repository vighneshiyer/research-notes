# Chisel Sequences

## 3/22/2023

- https://github.com/riscv-software-src/riscv-tests/tree/master/isa
    - You can build `make rv64ui-v-add`, which will link from `env/v` which contains a very small virtual memory implementation
    - Maybe try using `pk` with a regular userspace build of any C program and that program should use virtual memory by default since `pk` acts like a supervisor (running in machine mode)
- package.scala contains the top-level ADT for sequences and properties
    - toAutomaton converts that to the backend ADT (in Sequence.scala)
    - then each backend (Spot / SequenceFSMs) turns the backend ADT into an automaton module

```scala
// S <: Data = type with hardware bound
// UInt(8.W) = value of type UInt
// 100.U = literal value of type UInt
//
// SequenceIO[UInt](UInt(8.W))
// SequenceIO[S](genState=initialState = S <: Data)
```

## 3/15/2023

- Local variable support for Chisel (without Spot, but rather using the SequenceFsms backend)

```scala
class SequenceIO[S <: Data](gen: S) extends Bundle {
  /** input for property's current local state */
  val localState: S = Input(gen())
}
val bundleInstance = new SequenceIO[UInt](UInt(8.W))
```

```scala
class Example extends Module {
    val a = IO(Input(Bool()))
    val b = IO(Input(Bool()))

    // 1. Type-parameter free API, using magic state binding structure
    val state = ChiselStateBinding(Bool())

    PropSeq(
        SeqImpliesNext(
            SeqExpr(a, state.poke(a)),
            SeqExpr(b == state.peek())
        )
    ) // this may be tough to implement, need to pass the "state" along to the sequence interpreter

    // 2. Type-parameter API, state type is explicitly part of the Sequence type
    assertAlways(PropSeq[S=Bool](
        SeqImpliesNext[S=Bool](
            SeqExpr[S=Bool](_ => a, _ => a),
            SeqExpr[S=Bool](s => b == s)
        )
    ), initialState = false.B)
}
```

## 3/8/2023

- We discussed local variable support, and it turns out that it isn't doable with an optimized automata from Spot since we can't correlate a state+transition in the optimized automata to a local variable read/write in the original sequence
- The solution is to implement local variable support only for the SequenceFsms backend
- Next steps:
    - Familiarize yourself with how those automata work
    - Read this paper: https://ieeexplore.ieee.org/abstract/document/1487886
    - Attempt to implement local variable support by adding IOs to SequenceIO that represents a sequences' unique handle to its parent property's local variable
    - Now we can use the local variable in the atomic proposition and in the state update within SeqExprModule

## 3/1/2023

- We want to seperate model evolution from asserting properties on that model itself
- MagicPacketTracker
    - We could implement a similar mechanism over Scala sequences
    - Keep a counter that is just another sequence over enqs and deqs
    - Read from that counter when a enq happens, store the nElems and data locally per property instance
    - Decrement property local nElems on every deq
    - When nElems becomes 0, check that the data coming out
- Next idea we should take a stab on
    - Take the existing Chisel sequences API and add local variable support
    - For the RTL synthesis case, we can define the maximum number of properties in flight and then synthesize that number of instances
    - For the case when the number of properties may be unbounded, or we don't want to set a cap, then just synthesize the atomic propositions, and poke them outside so that software can assert the properties by creating many instances in memory - need some kind of DPI interface that we generate from within a Chisel module as a Verilog blackbox
- First step
    - Basic local variable support for Chisel sequences
- Let's first see what it take to get this to work, at least with the Spot backend
    - May have to add type parameters on Property and other subtypes of Sequence
    - May have to add an "initial" value of the local variable as an argument to assertAlways
- This API might be ugly
    - Another approach is to declare local state as a binding that can be referenced in scope within a property
    - No type parameters required, but some binding needs to be created out of the property context that can be "peeked" or "poked"

```scala
class Thing extends Module {
    val io = IO(new Bundle {
        val a = Input(Bool())
        val b = Input(Bool())
    })
    val propState = StateBinding(Bool()) // type: StateBinding[T <: Data](d: T)
    // for the fifo case
    // val propState = StateBinding(new Bundle {
    //     val nElems = UInt(8.W)
    //     val data = UInt(32.W)
    // })
    val prop = PropSeq (
      SeqImpliesNext(
          SeqStateExpr(a, propState.poke(a)),
          SeqStateExpr(propState.peek() === b)
      )
      // type: SeqStateExpr(prop: Bool, () => Unit = () => ())
    assertAlways(prop, propState???)
}
```

## 2/22/2023

- It is not clear whether this API is the best for asserting FIFO properties
    - Rather using a simple state folding + inline assertions on the scala sequence can work quite well
    - Thing to explore: how would this property be written with SVA instead?
- Does it make sense to externalize state update rules from the checking functions?
    - Tying state updates to the truthiness of a predicate makes it difficult to use the state update in a further check
    - e.g. if we update the state on a dequeue, then we want to check that the thing that was dequeued matches what we expect, but that happens after the state update (meaning we would have to carry over the dequeued data into the next atomic proposition)
    - If we make the state update something that happens in the background (just like synthesizable RTL), then we can write propositions that look for a dequeue happening, store the expected data from the background state update, then verify that it matches what we expect (in the subsequent property)

```
(action == dequeue(deqData), dActual = from the state update rule, deqData = from the AP) |-> (deqData == dActual)
```

- We want to be able to write unit tests for Chisel and Scala sequences outside of elaborating RTL
- We want to have direct control over the APs themselves

```scala
assert (a === 2.U) |=> (b === 3.U)
sealed trait AP
class ChiselAP(b: chisel3.Bool) extends AP
class ScalaAP(s: LazyList[Bool]) extends AP
def testAP(ap: AP): Boolean = {
    ???
}
```

## 2/15/2023

- All of the below are now fixed and implemented
- Spot integration for ScalaSeq's seems to work (no local variable support yet)
- Or and And parsing using Spot HOA emission works and produces correct automatons both for Scala and Chisel sequences
- Accepting states are no different than regular states for our problem
    - Treat them as just normal states that have transitions
    - Accepting states can indicate the "completion" of a property
- Properties should have a Globally around them G(...)
    - They are always checked in a continuous fashion, never just for the initial run of a trace

### Real-World Use Cases

- FIFO checker:
    - Can we write a FIFO specification using our sequences library?
    - This will involve the usage of local state
    - We want to show two potential implementations
        1. Pure Chisel implementation designed to be synthesized
        2. A Scala implementation that can take raw signal values from the testbench and check the same properties
    - Chiseltest testbench for a Queue
        - Drive the enqueing and dequeuing interfaces randomly or with some fixed pattern
        - Peek all the interface values every cycle
        - Write a (or more than one) ScalaSeq that checks the FIFO properties
            - Data integrity - if I write one piece of data, I want to get the same piece out
            - Data ordering - if I write two pieces of data, I want to get the first one out before the second one
            - May have to use some local state to mock the state of the FIFO as it evolves
        - Add local variable support for the Spot HOA ScalaSeq checker
        - Use Spot to convert the ScalaSeq's into functions we can call with concrete traces
            - Pass the traces from RTL simulation into the ScalaSeq checker
            - Verify there are no errors
    - An example of a FIFO checker: https://github.com/ekiwi/comparing-random-testing-and-bmc/blob/main/deepbug/src/deepbug/harness/FifoSoftwareHarness.scala

- Tilelink checkers:
    - https://github.com/tianrui-wei/assertions/blob/master/tilelink_checker.sv
    - https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tilelink/Monitor.scala
    - Q: Can we write these in our sequences language more succintly and potentially with better performance?

### Property Introspection and Debugging

- There is a lot of future work in making SVA style properties debuggable
    - Once we get started with improving the API and get some real world use experience with the FIFO/TL checker, we can work on this topic
    - TODO Vighnesh: publish my notes from SLICE winter 23 retreat so you can look at them

## 2/8/2023

- Actual fix is due to incorrect parens - here is the fix: a & X (b & X (c))
    - This is fixed
- Working on Spot integration for ScalaSeq's - working on bridge between HOA AP map and the ScalaSeq
- HOA parser question
    - All the boolean expressions are product of sums (abc) + (!bd) + (c!a)
    - I used a parser combinator library to write the parser for these expressions (https://github.com/com-lihaoyi/fastparse)
- Bug with Spot for Chisel seq that uses SeqOr
    - PSL: G(a | b)
    - HOA does it look right?
    - In Spot.scala, look at hoaString, hoa, and finally inspect the SpotPropertyAutomata and check its Chisel circuit for correctness

```scala
// What we expect for this G(a | b) HOA
automataState := 0.U
willFail := !a && !b
failed := Mux(failed, failed, willFail)
```

- Next things to check are sequences inside Or and And (rather than just simple predicates).

- Some reading here: https://ieeexplore.ieee.org/abstract/document/1487886

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

- TODO Vighnesh: rewrite the HOA parser so that everything is mapped on the original String values passed into SPOT as atomic propositions (rather than arbitrary integers that are assigned in the HOA header to each AP).
- TODO Vighnesh: rewrite the sequences frontend so that it can work universally with Chisel and Scala based sequences
    - Can we also unify the ADTs themselves?

```scala
sequence[Queue, Any] {
    val wr_ptr_max = AP(wr_ptr == queue.params.depth)
}
```

- TODO Vighnesh: gather a bunch of examples of temporal properties that we can use to benchmark the different implementations we have
    - Pure software impl - interpretation (with FFI)
    - Pure software impl - Java code generation and JIT compilation (with FFI)
    - SPOT -> HOA in software (with FFI)
    - Software impl -> convert to C++ -> compile with RTL simulator
    - Prop -> Chisel RTL evaluation in RTL simulation
- TODO: implement constructFormula
- TODO: use the map returned from constructFormula inside assertHOA to actually evaluate the HOA automata for a given trace
- Next step: figure out how to reintroduce local state for SPOT based automata checkers

## 11/16/2022

- Some fixes required for the assert tests (remember that asserts are checked globally just like covers so if there is a condition in the start of the assert it is checked on every element of the trace - there can be failures on the same timestep)
- TODO: try to consolidate the assert and cover functions (keep them separate, but the underlying functions they call should be very similar and just have flags that change their behavior slightly)
- TODO: state support
    - DONE! live
- Things to do:
    - TileLink / AXI4 monitor (https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tilelink/Bundles.scala)
    - SPOT integration
        - Look at `spot.scala`
        - `val psl = s"G(${sequenceToPSL(seq)})"`
        - `val hoaString = callSpot(psl)`
        - `val hoa = HOAParser.parseHOA(hoaString)`
        - ScalaSeq -> PSL/LTL string
        - Call spot
        - Get back the HOA automata
        - Write assert in terms of the HOA automata
            - assert(trace: T, hoa: HOA): AssertResult
        - ltl2tgba -B -D -f "G (a -> X b)"
        - Ignore state for now

## 11/9/2022

- Implementation of Or and Implies in the interpreter
- Prototyping per-property and global across-property state
- TODO Vighnesh: give push permission to Andy on chisel-sequences
- TODO: add assert function
- TODO: fix bugs in Or and add additional unit tests for those
- TODO: state implementation

## 11/2/2022

- Look through the implementation of Scala level sequences and the cover checker here (https://github.com/ekiwi/chisel-sequences/blob/main/test/scala_sequences/SequenceCoverSpec.scala)
- Be able to run all the tests
- Add some more tests, notice that there are several nonimplemented primitives in my implementation (e.g. Or, …) (look for “???”)
    - Implement Or and Implies and write tests for them
- Make sure you can use IntelliJ as an IDE (or vscode)
- TODO: prototype local state support for sequences (per-instance state for a given sequence)

```scala
sealed trait ScalaSeq[+T, +S]
case class AtmProp[T, S](pred: (T, S) => Boolean, update: (S, T) => S) extends ScalaSeq[T, S]
case class Fuse[T, S](seq1: ScalaSeq[T, S], seq2: ScalaSeq[T, S]) extends ScalaSeq[T, S]
```

## 10/26/2022

- https://github.com/ekiwi/chisel-sequences
    - Spot backend for Chisel sequences works (can generate a property checker by parsing the automata returned by Spot and turning it into a Chisel Module)
    - Goal is to integrate the Scala sequences ADT in this repo and be able to use Spot as a backend too
- Given transactions on the A and D channels of a TL interface, report any violations by checking properties on them
    - https://starfivetech.com/uploads/tilelink_spec_1.8.1.pdf
    - Goal eventually is rocket-chip TLMonitor replacement (https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tilelink/Monitor.scala)
    - https://github.com/TsaiAnson/verif/blob/master/tilelink/test/TLSLPropertyTest.scala
- OpenTitan properties: https://github.com/lowRISC/opentitan/blob/55bb57493e93c2a4508dbd20f1e75c67c55cd7e9/hw/ip/tlul/rtl/tlul_assert.sv#L319
- FIFO RTL test, enq/deq
    - val enqTxns = Seq[Transaction(data, timestamp) | Nop]
    - val deqTxns = Seq[Transaction(data, timestamp) | Nop]

```systemverilog
sequence enqFire enq.ready && enq.valid
sequence deqFire(d) deq.ready && deq.valid && deq.data == d
assert enqFire {d = enq.data} ##[1:*] deqFire(d)
```

- https://www.doulos.com/knowhow/systemverilog/sva-properties-for-pipelined-protocols/ (SVA local variables)

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

- Vighnesh: Pure Scala implementation of sequences over generic type T and a checker for those (without using Spot)
- TODO: continue working on the check function, and reference my code (TBD)
- Goal eventually is rocket-chip TLMonitor replacement (https://github.com/chipsalliance/rocket-chip/blob/master/src/main/scala/tilelink/Monitor.scala)
    - AXI protocol: https://developer.arm.com/documentation/ihi0022/e/AMBA-AXI3-and-AXI4-Protocol-Specification/Introduction/About-the-AXI-protocol?lang=en
- First open source sequence / temporal prop spec API in existence

## 10/12/2022

- Sequence tracking implementations
    - Manual tracking of state, duplicate sequence state every time the sequence is ‘started’, and track on a index-by-index basis as you traverse the list
    - Something more systematic: take your sequence description, convert it to a standard specification language (PSL, LTL), give that to a automata tool (e.g. spot), spot will give you an automaton, then you interpret that automaton in Scala
- https://spot.lrde.epita.fr/
    - https://spot.lrde.epita.fr/concepts.html (READ THIS)
    - PLAY here: https://spot.lrde.epita.fr/app/
- SVA
    - https://www.doulos.com/knowhow/systemverilog/systemverilog-tutorials/systemverilog-assertions-tutorial/

## 10/5/2022

3. Temporal property specification language
    a. LTL (G a -> X b)
    b. Atoms are: atomic propositions and operators on those APs
    c. SVA (SystemVerilog assertions)

- Instead of jumping straight to properties, let’s just check where sequences are present on a Seq[T]
    - https://github.com/ekiwi/chisel-sequences (we will get our code in here to merge with the Chisel sequences implementation - we want to unify the API for chisel and pure Scala sequences)

```scala
val isTrue = atomicProp[Bool](i => i)
val isFalse = atomicProp[Bool](i => !i)
val trueThenFalse = Concat(isTrue, isFalse) // one “cycle” between true and false
ConcatFuse(isTrue, isFalse) // both are true on the same cycle (never possible in this case, but useful for generic sequences over some T)

def check[T](seq: Sequence[T], trace: Seq[T]): (completed: Seq[(Int, Int)], pending: Seq[Int])

check(trueThenFalse, Seq(true, false, true, false, false, true)) == ([(0, 1), (2, 3)], [5])
```

- Play with more complex sequences on different types T and write unit tests for check
- Next week, we can move on to property definitions and unification with chisel sequences

## 9/30/2022

- Repo: https://github.com/ekiwi/chisel-sequences
- SVA vs PSL: PSL AND SVA: TWO STANDARD ASSERTION LANGUAGES ADDRESSING COMPLEMENTARY ENGINEERING NEEDS
- Unification of property APIs across testbenches and RTL
    - Same API can describe properties across Chisel Data and for Streams/Seqs of any Scala datatype
    - Ideally, a property should be testable outside a chiseltest/Chisel3 context
- Interpretation variants:
    - Direct SVA string emission in a Verilog blackbox
    - Testbench-connected software (Scala) interpretation / streaming Scala interpreter
    - RTL monitor emission via BlueSpec-like reduction
    - RTL monitor for formal (property start point becomes a free variable for the solver to choose)
    - Scala -> Native (machine code monitor) for linking within Verilator/VCS simulation
- Alternative backends (vs BSV technique)
    - SPOT (https://spot.lrde.epita.fr/)
- Other attempts of creating a property specification language
    - GHDL PSL
    - Yosys SVA to FSM
        - https://tomverbeure.github.io/rtl/2019/01/04/Under-the-Hood-of-Formal-Verification.html
        - Generating Hardware Assertion Checkers - Marc Boule
    - https://github.com/iscas-tis/CHA (Chisel SVA attempt)
    - Investigate if anything exists for SpinalHDL
- TODOs:
    - User-facing API to construct sequences/properties
    - Can we describe recursive properties / sequences?
    - Better testing (unified API for interpreted, synthesized, code gen’ed, etc. properties)
