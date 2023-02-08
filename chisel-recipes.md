# Chisel Recipes

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
