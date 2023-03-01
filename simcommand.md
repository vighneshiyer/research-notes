# SimCommand

## 3/1/2023, Wed

- NeuromorphicProcessorTB - now its about 300 kHz w/ Command and 450 kHz with single-threaded chiseltest
- DecoupledGCD
    - For the Command case, there is substantial overhead in implicit conversion of Data to testableData for peek
    - My instinct: peek on its own should be fast, but if the Data has many fields, then we can burn time accessing the Map[String, Data] in the Record type
    - But, this should only be an issue with smaller circuits where the step() time doesn't dominate

- https://www.chisel-lang.org/chisel3/docs/appendix/experimental-features.html#bundle-literals-

```scala
import chisel3._
import chisel3.experimental.BundleLiterals._

class MyBundle extends Bundle {
  val a = UInt(8.W)
  val b = Bool()
  def lit(a: UInt, b: Bool): MyBundle
}

class Example extends RawModule {
  val out = IO(Output(new MyBundle))
  out := (new MyBundle).Lit(_.a -> 8.U, _.b -> true.B)
  val v = new MyBundle
  v.a := 8.U // THIS CAN'T BE DONE IN A OUT-OF-MODULE context
}

trait Interactable[I] {
    def set(value: I): Unit
    def mutate(value: I => I): Unit = (i) => i.Lit(_.fieldA -> 1.U)
    def get(): I
    def compare(i: I): Command[Boolean]
}
```

- https://hackage.haskell.org/package/clash-prelude-1.6.4/docs/Clash-Explicit-Signal.html#t:Signal
    - Signal is a monad and we can 'unpack' values in a signal and use them in other expressions that yield a Signal
    - This builds a circuit without any 'global' builder context
    - Clash is a custom compiler (not a deep embedding / DSL)
- Another approach that uses a deep embedding in Haskell: https://github.com/blarney-lang/blarney

## 2/24/2023, Fri

- async-profiler, use with IntelliJ plugin (sbt is annoying)
    - ran the GCD benchmark with treadle - found vast majority of time is spent inside peek() function
        - we found this is because the default backend is treadle
    - pending profiling run on Verilator
- switch to pure mill build system (get rid of sbt)
    - https://com-lihaoyi.github.io/mill/mill/Plugin_Jmh.html
- prototype the "stateful box" that can be used to have Commands drive other Commands
    - Interactable datatype looks good
    - Next steps: add Interactable in UARTCommands, attempt to actually test 2 commands with each other
- Threads now have an extra order parameter (just an integer), just a stopgap measure
    - Can use this to get deterministic behavior in the prescence of combinational loops between fork blocks
    - WIP: the MockTest that shows commands testing each other, check that time travel works correctly
- WIP: detect combinational loops between two or more forked blocks that read/write to the same Interactable

## 2/14/2023, Tues

- All the 'Quick Stuff' below is done and merged!
- The benchmarks for tailRecM vs the primitive are also in, but awaiting rebase
- Also the "time-travel" stuff is also in - we need a decent benchmark for this
    - TODO: refactor the UART receiver so that it doesn't poll every cycle
    - It looks like the singlethreaded chiseltest backend actually does pass through multi-cycle steps directly to the Verilator so, so we should expect performance gains from this
- TODO: look at the JMH tests, make sure the benchmarks make sense and record the performance numbers
- TODO: a real test with a real DUT
    - riscv-mini (is a potential target)
    - NeuromorphicProcessor
- TODO: experimentation with unit testing VIPs / Commands
    - Maybe also be able to test Commands with Commands (have bindings that both Commands are connected to)
- Priority and monitor regions
    - We haven't still discussed how to resolve inter-thread race conditions
    - If two threads are poking and peeking the same net in the same cycle, we really should raise a runtime exception
    - To mitigate this, one or more threads can be designated as a monitor, in which case it will always run last
    - Solution! - just use fixed point iteration
    - Software analog for Pokeable is an MVar or some STM stuff??? I'm really not sure about this.

```scala
sealed trait Pokeable[P]
case class RawPoke[V](var initialVal: V) extends Pokeable[V]
case class ChiselPoke[B <: Data](IObinding: B) extends Pokeable[B]
def poke[P](p: Pokable[P], data: P) = {
    p match {
        case r: RawPoke => r.initialVal = data
        case c: ChiselPoke =>
            import chiseltest._
            c.IObinding.poke(data)
    }
}

val a = RawPoke(chisel3.Bool())
val p = for {
    _ <- poke(a, 1.B)
    _ <- peek(a)
    _ <- writetoChannel(c, ...) // tbd, this seems to complicate matters a lot
    _ <- step()
    _ <- poke(a, 0.B)
}
val q = for {
    v <- peek(a)
    _ <- expect(v == 1.B) // this will fail, b/c Chisel bundle literals don't check literal equality
    _ <- step()
    v2 <- peek(a)
} yield (v, v2)
val pq: Command[Unit] = for {
    _ <- fork(p)//, FIRST)
    _ <- fork(q)//, SECOND)
} yield ()
val r = unsafeRun(pq, ???, config)
```

## 1/31/2023, Tues

- Continued work on combinators
- Moved tailRecM into Command
- GPU accelerated RTL simulation
    - https://www.design-reuse.com/industryexpertblogs/39855/what-is-rocketsim-why-did-cadence-acquire-rocketick.html
    - https://research.nvidia.com/publication/2022-08_rtl-cuda-gpu-acceleration-flow-rtl-simulation-batch-stimulus
- TODO:
    - Continue to pick off the quick tasks, bump when your PR is ready and we'll merge
    - Vighnesh: come up with a more comprehensive testbench environment (e.g. for a Rocket core) to use with SimCommand (maybe riscv-mini is a good initial target)

## 1/24/2023, Tues

### Classes

- CS 152 (likely drop), CS 126 (probability), CS 184 (graphics), CS 189 (ML), CS 270 (algorithms)
- EE 219C (formal methods) (https://people.eecs.berkeley.edu/~sseshia/219c/)
    - Modern formal modeling research:
    - https://ieeexplore.ieee.org/abstract/document/9218715 (A-QED Verification of Hardware Accelerators)
    - https://dl.acm.org/doi/abs/10.1145/3282444 (Instruction-Level Abstraction (ILA): A Uniform Specification for System-on-Chip (SoC) Verification)
- Take max 3 classes this semester, CS 189 isn't really necessary

### SimCommand Engineering

#### Quick Stuff

- SimCommand PR (https://github.com/vighneshiyer/simcommand) - merge your PR after resealing Command
- Fix up the combinators
- Benchmark of recursion primitive vs tailRecM for some simple task like waiting for a signal to go high but it takes 100k cycles
- moving tailrecM as a Command method
- Step returns time after the step
    - or a Info primitive (if we determine that other simulation global info is useful)

#### A Bit Harder

- implicit conversion of Command[Boolean] to a special Command class that has a ifM method
- read vs read/write threads with a value encoding (rather than type encoding)
- Benchmark of imperative vs recursive interpreter (use jmh)
    - Profile both interpreters for low hanging fruit (https://github.com/jvm-profiling-tools/async-profiler)
- Opportunistic time travel - when 2 threads are waiting on different amounts of cycles, can we just travel to min(cyc1, cyc2)
    - Transactors (Accelera spec called SCE-MI)

#### Even Later

- Create plain Verilog testbenches using record and playback from the SimCommand interpreter
- Benchmark data structures in SystemVerilog vs Scala (dicts, lists, fixed width arrays)
- Logging / error / warning primitives in the Command ADT (https://github.com/zio/zio-logging)
    - Separate logging backend from frontend (frontend is just a simple Log primitive + verification specific limits, backend can be log4j or others)
    - Add an 'expect' primitive to make chisel Bundle literal comparisons easier

### Experimental

- Unit testing VIPs
    - Peekable / Pokeable typeclass (Chisel signal in testbench environment - chiseltest closure, user controllable signal)
    - Ability to unit test Commands without having to elaborate any RTL or use checkers that must be implemented as RTL
    - Later goal: test Commands with Commands

```scala
val pokeBind = Binding()
val cmds = UARTCommands(tx=pokeBind)
test(cmds.sendByte(Seq(1))) {
  assert(pokeBind == 0)
  advance 10 cycles
  assert(pokeBind == 1)
}
```

### Future Looking

- Design of a new RTL simulator API (independent of Chisel) (but with a good bridge library)
    - Use Java 19 FFM
    - Investigate chiseltest VirtualThread vs Thread
- GraalVM, Graal AOT compilation

### Vighnesh TODOs

- TODO: More extensive tests that are realistic (AES/SHA3 accelerator)
- TODO: grand unifying vision of what SimCommand ought to be
    - SystemVerilog replacment for verification (SVA, FSM construction library)
    - No compromise on perf or features
    - Integrate all the work that everyone is doing

## 12/1/2022

- Primitive in Command ADT for tail recursion
    - ZIO and cats IO seems to have something similar
    - Idea: create a ADT case for generic recursion, compare its performance for implementing repeat/concat against tailRecM (trampolining)
- Extend Command ADT with Kill
- Can we emulate join_any using the existing threading join/kill constructs?
    - Do we need a special ADT case for this?
    - What are the common use cases of join_any besides timeouts.
        - How do we implement timeouts using our existing API (if even possible)
- Do we need a special join_all primitive, or can we just join a bunch of threads one-by-one instead? What is the performance overhead? What is typically done in a testbench to warrant this behavior in the first place?

## 11/17/2022

- TODO: investigate why enqueueNow in chiseltest uses a fork in the monitor region to wait for ready to go high (instead of doing it synchronously)
    - QueueTest may have an answer
    - Prototype using the type system to track commands that are read-only vs read-write and automatically schedule the subcommands in the correct region within the interpreter (runtime type tag)
- `.asInstanceOf[R]` is sometimes required (https://github.com/bdngo/chisel-fifo/blob/main/src/main/scala/interpreter/Interpreter.scala#L33)
- TODO: construct testcase for deadlock on multiple joins
- https://docs.cocotb.org/en/stable/triggers.html#cocotb.triggers.First (join_any variant in cocotb)

## 11/9/2022

- Ruled out usage of scala-async due to HOF issues and slightly worse performance
- Still unsure about JMH benchmarking results (captures overhead of chisel compilation and treadle rather than async/await)
- TODO Vighnesh: how often are mutexes or semaphores or mailboxes really used in SystemVerilog - find some open SV testbenches and inspect them
    - Wavious
    - Intel
- TODO Vighnesh: GET THE SIMCOMMAND REPO CLEANED UP AND PUBLISHED BY FRIDAY

## 11/2/2022

- scala-async performance benchmarking
    - async in HOFs is janky
    - Benchmarking in Scala (JMH) - https://github.com/sbt/sbt-jmh
    - https://www.gaurgaurav.com/java/scala-benchmarking-jmh/
- Channels
    - https://www.chipverify.com/systemverilog/systemverilog-mailbox
    - Like gochannels
- Interface type parameter
    - Interface as trait (look we’re getting quite similar: https://zio.dev/overview/)
- cocotb coroutine synchronization primitives https://docs.cocotb.org/en/stable/triggers.html
- SystemVerilog primitives
    - https://www.chipverify.com/systemverilog/systemverilog-mailbox
    - https://www.chipverify.com/systemverilog/systemverilog-semaphore

```scala
test(new Peekable()) { c =>
c.a.poke(10.U)
c.a = 10.U
```

Note, these lines are different. The first uses ‘c.a’ just as a reference so chiseltest knows which port to poke in the RTL simulator. The second is just illegal since ‘c.a’ is a val. Critically, the Scala object representing the Chisel circuit is never ‘updated’ via peeks and pokes - it is only used as an ergonomic reference!

Can we avoid regions, by using type parameters to mark commands which are read only vs readwrite? Sure, but it may not be worth it.
Later: this can be accomplished more ergonomically with dependent types rather than using ADTs to encode type-level constraints in type parameters.

```scala
sealed trait Behavior
case object ReadOnly extends Behavior
case object ReadWrite extends Behavior
sealed abstract class Command[+R, I, +B <: Behavior]
protected case class Poke[I <: Data](signal: I => Data, value: I) extends Command[Unit, R, ReadWrite]
```

How could an interface type parameter work?

```scala
sealed abstract class Command[I, +R]
protected case class Poke[I <: Data](signal: I => Data, value: I) extends Command[I, Unit]
protected case class Peek[I <: Data, V](signal: I => Data) extends Command[I, V]
// use the I type parameter in peek and poke
```

Example usage:

```scala
Poke[DecoupledIO[UInt]](intf => intf.valid, true.B)
Poke[DecoupledIO[UInt]](intf => intf.bits, 100.U)
```

Usage with stand-in mocked model vs running RTL simulation

```scala
def runTest(command: Command[R, I], model: Model[I]): R
trait Model[I] {
    def receivedPoke(signal: I => Data, value: Data): Unit
    def resolvePeek(signal: I => Data): Data
}
class QueueModel(gen: DecoupledIO[UInt]) extends Model[DecoupledIO[UInt]] {
    val bundle = gen
    var bits: Int = 0
    def receivedPoke(signal: DecoupledIO[UInt] => Data, value: Data): Unit = {
        Map[Data, Data => ...] // this could be a more reasonable user-facing API
        if (signal(bundle) == signal.bits)
           { bits = value }
    }
    def resolvePeek(signal: I => Data): Data
}
```

The real API for RTL simulation

```scala
def runReal(command: Command[R, I], intf: I): R
test(new Queue()) { c => runReal(poke, c.enq) }
```

Use a command to test another:

```scala
class If extends Bundle { val a = Bool() val b = Bool() }
test(If()) { c=>
    val prog1 = for { _ <- poke(c.a, true.B) p <- peek(c.b) } yield p
    val prog2 = for { _ <- poke(c.b, true.B) p <- peek(c.a) } yield p
}
```

HOW?! We will have to resolve ordering of ‘bridge’ 2 commands. Nice property is that a bridged command has no interface requirement (see the Any type parameter).

```scala
def bridge(cmd1: Command[I, R1], cmd2: Command[I, R2]): Command[Any, (R1, R2)]
```

How is a channel actually used? Monitors are a good example:

```scala
def monitor(intf: DecoupledIO[I], txChan: ChannelHandle[MonitorTx[I]]): Command[ThreadHandle[Unit]] ={
    val receiveOne: Command[MonitorTx[I]] = for { // something }
    val prog = for { tx <- receiveOne _ <- put(tx, txChan) }
    for { handle <- fork(forever(prog)) } yield handle
}
```

Then the thread that created this monitor can kill it once it is no longer needed - it can also poll on the txChan handle to process transactions this monitor received from the DUT

## 10/25/2022

- scala-async

```scala
peek(Data): Data
poke(Data, Data): Unit
step(Int)(implicit scheduler): Future[Unit] = Future.from(clock.step() // chiseltest step call)

step(1).get() // blocking call
step(1).andThen(() => peek, poke, … step(1).andThen(() => …): Future[T] // “lazy”, callback hell approach (just like in JS)
await step(1); peek, poke, … ; await step(1); … // promises-like approach (under the hood scala-async will make this into nested callbacks)
```

- Prototype: types as above, straight-line simulation function that just interacts with the DUT (e.g. ALU, just poke and peek numbers for it to add, and step and check with expect)
    - Comparisons: single-threaded chiseltest natively, Future with scala-async, Command monad
- Working kill primitive
    - I have a good use case for this (2 thread, sender and receiver operating on a DUT, and the testbench master thread should kill the receiver after the sender has sent its payload)

```scala
def repeat(c: Command[T], n: Int): Command[Unit] = tailRecM
def repeatCollect(c: Command[T], n: Int): Command[Seq[T]] = tailRecM
case class Repeat(c: Command[T], n: Int) extends Command[Seq[T]]
case class For(s: S, c: S => (Command[T], S), done: S => Bool) extends Command[S]
def concat(c: Seq[Command[T]]): Command[Seq[T]] = c.sequence
```

- You can implement Traverse for Command (https://typelevel.org/cats/typeclasses/traverse.html) and sequence comes for free
- https://github.com/oleg-py/better-monadic-for
- Long term:
    - SLICE Retreat (Winter 23) - early /mid Jan (few days before start of semester)
    - Poster / segment of a talk
    - Intel and Amazon both “use” Chisel - they also use chiseltest - any performance win is great news for them (as long as its usable)
    - Things that will be useful
        - Published SimCommand repo that is easily usable with the upstream chiseltest
        - More benchmarks proving performance improvements
        - Performance not being limited by the monadic interpretation overhead (perf parity with straight line chiseltest code / emulated threading)
        - Multithreading of simulation threads
            - Some threads are “read-only” and they can possibly operate on a different core that can run behind the “read and write” threads
        - Thread order dependency (catching potential non-determinism issues in the interpreter)
        - Kill primitive (and determinism) and why that doesn’t exist in SystemVerilog
        - Testing Command’s without using RTL simulation (pure software)
            - `Command[I, R]`
            - Currently: `def drive[T <: Data](data: T, intf: DecoupledIO[T]): Command[Unit]`
                - `test(new Queue(...)) { c => val x = drive(100.U, c.enq) // test the drive function }`
            - `def drive[T <: Data](data: T): Command[DecoupledIO[T], Unit]`
                - `test()`
        - Testing Command’s with Command’s (mocking hardware)
        - Commands used for hardware description and high level synthesis
            - `Command[R].flatMap(_ => Command[_])`
            - `SynthCommand[R <: Data].flatMap(Data => SynthCommand[Data])`
            - `SC(10.U).flatMap { n => {if (Random.nextBool()) lift(n + 10.U) else lift(200.U)} }`
            - `class X extends Module { if (random…) val x = 100.U else val x = 200.U }`
        - VIPs for real interfaces (e.g. AXI4: https://developer.arm.com/documentation/ihi0022/e/AMBA-AXI3-and-AXI4-Protocol-Specification/Introduction/AXI-Architecture?lang=en)

## 10/12/2022

- scala-async prototyping
    - Can we override the executorcontext?
- kill construct
    - We want this to be deterministic, so only kill a thread after all other threads have finished for the given clock cycle
    - Current implementation: add thread to kill list and then kill after all the threads have completed for that clock cycle
- TODO: SystemVerilog example of non-deterministic (i.e. thread spawn order dependent) kills vs our deterministic implementation (to show our version is superior)
- Benchmark different thread yielding methods
    - Monadic flatMap composition
    - scala-async
    - ZIO to implement step / fork / join / … (this may be impossible to do deterministically)
- Performance improvement for primitive constructs for looping
    - `waitUntil(b: Command[Boolean], body: Command[T])`
        ▪ Is this stack safe?
        ▪ If it is stack safe, is it probably not performant
    - Can we replace this with a primitive looping construct as part of the Command ADT?
    - Goal: perf parity with single-threaded chiseltest

## 10/5/2022

2. SimCommand (monadic testbench library with fork/join support)
    a. Command[R] is a description of how to execute a testbench which interacts with an RTL simulator
    b. An interpreter actually runs the description

- SimCommand feature completeness and production quality (Young-Jin)
    - port the chiselverify imperative runtime to Simcommand
        - https://github.com/yjp20/chiselverify
        - https://stackoverflow.com/questions/59176271/how-can-i-implement-loops-that-dont-use-potentially-very-large-amounts-of-heaps
    - Vighnesh: clean up SimCommand
    - add debug primitives
    - add peek/poke capturing as an interpreter option to simplify unit testing
    - prototype inter-thread communication channels
        - what should the primitives be?
        - how can we make sure that the thread evaluation order doesn’t affect the order / cycles in which messages are sent/received
        - consider a kill primitive
    - how can we maintain state within a thread?
        - Should a state monad be a part of the Command type signature?
        - e.g. I want to have thread local state that I can update in ‘imperative’ style (without using recursion)
        - see Ref (in ZIO)
    - Prototype including the RTL interface type as a Command type parameter - how does it enhance testability of Command outside the chiseltest context? Does it come with other typing burdens?
    - Implement step as an async call (that calls chiseltest's step) and try to implement a straight-line test using chiseltest. Measure the overhead of Future. Does this approach beat the monadic approach? Certainly it is easier to write for people new to FP. (use scala-async)
    - We should prototype an implementation of For or While (or the most general looping construct you can think of) as a Command primitive (see similar ideas in JAX https://jax.readthedocs.io/en/latest/notebooks/Common_Gotchas_in_JAX.html#control-flow)
        - How does this interact with (or use) a built in state monad in Command?
