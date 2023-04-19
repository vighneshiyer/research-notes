# Chisel Recipes

## 4/19/2023

- We need to concretely define what go/done/active actually means
  - In particular, we need to express what it maps to in the RTL-level implementation and examples of how it would be used in a real recipe to do some real task
  - The question is whether active should be high when done is also high, or if it should fall one cycle before (this is the current behavior)
- Try to debug the DecoupledGCDSpec again - looks like something is broken with the DoWhile construct
  - We notice that the `go` signal doesn't go high at the start of the recipe, but rather somewhere in the middle
- One first step is to debug the doWhile primitive in isolation for a bit (look at the waveform, also add the active signal as a named signal in the VCD so we can verify its correct behavior)

## 4/12/2023

- Differences
    - Handwritten input.ready is high immediately after reset, whereas the recipe version sets input.ready on the same cycle when input.valid is high
    - The recipe version output.valid is delayed for one cycle compared to the handwritten version
- One issue is that active is only high in the doWhile combinator when the 'while' portion is happening (not in the first iteration of the loop).
- Solution: add an active signal for every primitive module
    - Modify Recipe.scala, now each recipe does (go: Bool => done: Bool, active: Bool)
- Other issue is too many dumb wires are created with similar names to the important ones
    - Solution: turn these wires into temporaries rather than named wires inside each recipe module

## 4/5/2023

- Working on fixing the formal equivalence test - rectify using doWhile
- Figure out how to make the DecoupledGCD recipe equivalent to the original DecoupledGCD
    - This involves driving input.ready to true *before* input.valid is true (just like the original)
    - After this, the two circuits should be sequentially equivalent

```diff
   forever (
-    waitUntil(input.valid),
+    waitUntil(input.valid), // in the original, ready is true immediately, not gated on valid
     action {
-      val bundle = input.deq()
+      val bundle = input.deq() // ready is only true after input.valid is true
```

## 3/23/2023

- Got a basic strict I/O equivalence formal test working with DecoupledGCD (handwritten against the recipe version)
    - Found a mismatch on the initial value of ready - this is a true mismatch
    - This can be rectified by using doWhile

## 3/15/2023

- Write a unit test that has a whileloop whose body takes multiple cycles and we should validate that 'active' stays high for the entire duration of the loop and body
- Attempt to write a doWhile combinator in terms of the whilePrim primitive
- Make whilePrim private
- Formal verification of equivalence between recipe and hand-written implementations
    - https://github.com/ucb-bar/chiseltest/blob/main/src/test/scala/chiseltest/formal/examples/DecoupledGCD.scala#L13
    - Idea: create a module that instantiates two versions of the GCD module (one recipe and one hand-written)
    - Assert every output should be equal between the two instances
    - Tie the inputs of both of them together
    - Then use BoundedCheck to prove that they are equivalent
    - Watch this: https://www.youtube.com/watch?v=ssAbq5tdh8Y
    - Read this: https://github.com/agile-hw/lectures/blob/main/22-formal/lec22-formal.ipynb
- Pull in rocket-chip as a Maven dependency and implement the TLRAM using recipes

## 3/8/2023

- We rewrote the DecoupledGCD recipe as imperative logic
- But we noticed that there is extra state that doesn't really need to be there (e.g. we can reuse a tick register with another tick that is exclusive, but this is a whole-program optimization, that may not make sense anyways)
- Next step:
    - We want to expose an 'active' signal that a user can tie to so that they know when each recipe block is active
    - The 'active' signal can be used to drive top-level I/Os as to minimize the amount of explicit state the user need to construct and have control over (e.g. resultValid register in decoupled GCD recipe).
    - Refactor both GCD and decoupledGCD to use this feature
- TODO: DoWhile primitive (we can just repurpose While) (https://docs.scala-lang.org/scala3/reference/dropped-features/do-while.html)
- TODO: figure out why debug wires for While aren't being named properly
- TODO: think about a good standalone example for recipes
- TODO: do a PPA (not power) comparison (just using yosys) of the recipe and raw version of the decoupled gcd module
- TODO: support all the other primitives that Blarney does (Fork, Background, Parallel, Dispatch) + multiple if-elseif-else block support and recipe delegation

## 3/1/2023

- The while recipe could be simplified by looping the body's done signal either back to the go signal of the body or to the done signal of the entire while recipe
- Testing the while recipe module in isolation to make sure the done signal goes high when we expect and is only a single cycle pulse
- ITE: should the semantic be that ITE is only active when go is true or should it be continuously active?
- Write a WaitUntil recipe in terms of While (the body is just a Tick, and the condition is the boolean thing we want to wait until)
  - Write a Forever recipe in terms of While
- Rewrite the GCD recipe as imperative code:

```scala
Forever { // implicit time stepping
  WaitUntil(io.loadingValues === true.B) // implicit time stepping
  Action {
    io.outputValid := 0.B
    x := io.value1
    y := io.value2
  }
  Tick
  While(y =/= 0.U) {
    when(x > y) {
      x := x - y
    }.otherwise {
      y := y - x
    }
    Tick
  }
  Action {
    io.outputGCD := x // these assignments should be sticky (e.g. they should assign to a register)
    io.outputValid := 1.B
  }
}.compile()
```

- Next, try to write the DecoupledGCD in the same fashion

## 2/22/2023

- We worked on the code, got things to kind of work, but the compiler still needs work

## 2/15/2023

- We discussed a generalized way to 'compile' a Recipe by converting it into a RTL block that has 2 IOs
    - `go` a single cycle wide pulse input that tells the Recipe to begin execution
    - `done` another pulse output that tells when the Recipe has completed
- Problem:
    - propagation of `go` needs to be unique in time as it goes through overlapping Recipes
    - every tick go/done block consumes 1 register, which is excessive if you have many ticks (all of them can be collapsed into one register if they are all in the same Sequential block)

## 2/7/2023

- Spot automata library: https://spot.lre.epita.fr/
    - Can use it to construct monitor automata for temporal properties (LTL)
    - Take an existing automata (deterministic, non-deterministic) and optimize to a deterministic automata that has fewer states than what we started with
- Paso - imperative test description for transactions using chiseltest-like imperative API that desugars to an automata that is put along with the RTL in a formal testharness
    - https://github.com/ekiwi/paso/blob/main/test/examples/SerialAluSpec.scala
- Repo: https://github.com/bdngo/chisel-recipes
- Some issues regarding 'clk' vs 'clock'
- We want `compile` to turn the Recipe into a state machine

```scala
  val r: Recipe = Sequential(Seq(
    Action {
      () => {
        io.a := 10.U
      }
    },
    Tick,
    Action {
      () => {
        io.a := 0.U
      }
    },
    Tick,
    Action {
      () => {
        io.a := 20.U
      }
    }
  ))

  // compile should emit this code:
  val stateReg = RegInit(UInt(2.W), 0.U)
  stateReg := Mux(stateReg === 2.U, 2.U, stateReg + 1.U)
  when(stateReg === 0.U) {
    io.a := 10.U
  }.elsewhen(stateReg === 1.U) {
    io.a := 0.U
  }.elsewhen(stateReg === 2.U) {
    io.a := 20.U
  }
  io.a := DontCare
```

## 2/6/2023

Notes from discussion with Kevin:

- Build the Blarney Recipe automaton and pass it to SPOT for optimization
- Look at paso for imperative test spec to automata compiler (but optimization is tricky here), Kevin suggests using Spot or another external optimizer
    - for separate threads, make them seperate state machines instead of combining them into one + optimization for state saving

## 2/1/2023

- If interested, maybe work on an abutment flow for Hammer

- DecoupledGCD rewritten in Recipe-style
    - the updates to x and y are actually continuous, until y === 0

```scala
Sequential(
  WaitUntil(io.valid),
  ITE(io.load, ifTrue=Seq(
    Action {
        x := io.a
        y := io.b
    },
    Tick()
  ), ifFalse=Seq(Action {
    when(x > y) {
        x := x - y
    }.otherwise {
        y := y - x
    }
  }, Tick())
)


While(!io.valid,
Seq(
  List(
    If(io.load,
      Seq(
        List(
          Action(x := io.a),
          Tick(),
          Action(y := io.b),
          Tick(),
        )
      ),
      If(x > y,
        Seq(
          Action(x := x - y),
          Tick(),
        ),
        Seq(
          Action(y := y - x),
          Tick(),
        )
      )
    ),
    Tick(),
    Action(io.out := x),
    Tick(),
  )
)
)
```

- A simple counter


```scala
class Counter extends Module {
  val io = IO(new Bundle{ val out = UInt(8.W) })
  val r = Reg(UInt(8.W))
  io.out := r
  r := r + 1.U

  Forever(
    Action { r := r + 1.U },
    Tick()
  ).compile()
}
```

## 1/25/2023

- Create a starter repo for chisel-recipes
    - Use a Github org

```scala
class Thing extends Module {
    val io = IO(new Bundle {
        val a = Output(UInt(16.W))
    })
    // State machine
    // 1. output 10
    // 2. output 0
    // 3. output 20

    // class State extends Enum { STATE_0, STATE_1, ...}
    sealed trait Recipe
    case class Sequential(s: Seq[Recipe]) extends Recipe
    case class Action(f: () => Unit) extends Recipe
    case object Tick extends Recipe

    val r: Recipe = Sequential(Seq(
        Action {
            io.a := 10.U
        },
        Tick(),
        Action {
            io.a := 0.U
        },
        Tick(),
        Action {
            io.a := 20.U
        }
    ))

    def compile(r: Recipe): Unit = {
        // use 2 passes over the Recipe, first to determine the transitions and required states
        // second to actually construct the circuit
        val state =
    }
}
```

## 11/30/2022

- TODO: devise a very simple state machine as a Chisel module that you write manually
    - Look at the GCD module for example
- Express the same state machine using the Blarney ADT (Recipe)
- Figure out if there’s a way to compile a Recipe down to a Chisel Module
    - To get the Chisel module data values that the Recipe can assign to or read from, we can use a similar technique to that here (https://github.com/ekiwi/chisel-sequences/blob/main/src/sequences/toAutomaton.scala)
    - We just extract a Map[String, Data] (something like this: https://github.com/ekiwi/chisel-sequences/blob/main/src/sequences/toAutomaton.scala#L26)
