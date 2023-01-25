# SimCommand

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

- Benchmark data structures in SystemVerilog vs Scala (dicts, lists, fixed width arrays)
- Logging / error / warning primitives in the Command ADT (https://github.com/zio/zio-logging)
    - Separate logging backend from frontend (frontend is just a simple Log primitive + verification specific limits, backend can be log4j or others)

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
