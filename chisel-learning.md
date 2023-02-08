# Learning Chisel

## 2/7/2023

- https://github.com/freechipsproject/chisel-bootcamp

```scala
def test[M <: chisel3.Module](mod: () => M)(testBody: M => Unit)
test[MyOperators](new MyOperators) // notice the type parameter is inferred, thanks to the 2 parameter lists
Seq[Int].fill(100)(2) // same here, the type parameter is inferred
```

- Why multiple clocks?
    - Off-chip IO, clock gating

- Can you use `val` to define a function?

```scala
val f: Function1[Int, Int] = (x: Int) => x + 2
// equivalent in Python: f = lambda x: x + 2
f(20)
def f(x: Int): Int = x + 2
f(20)
def g(v: Int, fn: Int => Int): Int = fn(v)
// equivalent in Python: g = lambda v, fn: fn(v)

val k = f(4) // "likely" compile-time evaluated expression
```

### TODO

- Look at the chisel-tutorial (https://github.com/ucb-bar/chisel-tutorial/tree/release/src/main/scala/problems)
- Get it working on your computer
- Implement a Queue in Chisel (without using any of the Chisel stdlib components)
- Test it using chiseltest.
- I want to see you use `sbt` and run your unit tests to verify the queue works.
- If you want, you can re-implement the tests from the 151 FPGA labs (for the FIFO).

## 10/20/2022

- Basic stuff wrt Chisel and chiseltest
    - edge detector
    - FIFO design and implementation
- Sungwoong: https://github.com/coltonha/template
    - `testOnly <package>.<class>`
    - gtkwave (to see vcd file in test_run_dir)
    - TODO: edge detector (Done)
- Grace: https://github.com/graceCXY/chisel
    - TODO: FIFO / memory
    - Questions:
        - FIFO understanding? I’m still not sure how gtkwave works/helps?
        - Downloaded verilog stuff… I think I understand but I don’t understand how it ties in to everything else?

## 10/13/2022

- Everyone: push your chisel-template forks to your Github account and share the repo links here (and bump me on Slack)
    - I’ll take a look and help you patch up the code
- Grace
    - Continue with Chisel prototyping (see the memory section and the FIFO)
    - Send the Slack group any questions you might have
- Yi
    - Also try to implement the FIFO in Chisel

## 10/3/2022

- Subword assignment is not allowed in Chisel - alternative to use a vector of Bool (if absolutely necessary)
    - See this: https://www.chisel-lang.org/chisel3/docs/cookbooks/cookbook.html#how-do-i-unpack-a-value-reverse-concatenation-like-in-verilog

```scala
case class AddrAndData extends Bundle { val data = UInt(8.W), val addr = UInt(16.W) }
val x = AddrAndData()
x.data := 20.U
x.addr := 10.U
val y = x.asUInt() // a bitvector that contains the address and data concatenated together as one unsigned integer
val z = y.asTypeOf(AddrAndData())
z: AddrAndData, z.data, z.addr
```

- Subword assignment hack (only if absolutely necessary)

```scala
val x = UInt(32.W)
val y = x.asTypeOf(Vec(32, Bool())
y(0) := …, y(1) := …, …
val z = y.asTypeOf(y)
```

- Everyone (use the chisel-template on your laptop, run sbt locally)
    - Implement a counter (https://github.com/EECS150/fpga_labs_sp22/blob/master/lab2/src/counter.v) in chisel and write a simple testbench in chiseltest
    - Implement an edge detector (https://github.com/EECS150/fpga_labs_sp22/blob/master/lab3/src/edge_detector.v) (see the spec for details)
    - Look at Chapter 6.4 (http://www.imm.dtu.dk/~masca/chisel-book.pdf). Design and test a memory.
    - Implement the FIFO in chisel (https://github.com/EECS150/fpga_labs_sp22/blob/master/lab6/src/fifo.v), reference the spec (and also the lab 5 spec, which covers ready-valid) (https://github.com/EECS150/fpga_labs_sp22/blob/master/lab5/spec/spec.md#ready-valid-interface)
- Everyone (see instructions above)
    - Collect coverage from running a testbench on Verilator
    - Make sure you can run a Verilog testbench using iverilog
    - Make sure you can use verilator and iverilog from chiseltest
