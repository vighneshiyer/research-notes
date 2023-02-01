# Chisel Recipes

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
