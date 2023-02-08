# Synthesizable VIPs

## 11/9/2022

- Firstly implement an ADT for Calyx IR itself and be able to generate Calyx IR and use the Calyx compiler to turn it into Verilog (and use chiseltest blackbox) to test it
- Simultaneously work on the synthesizable VIP for DecoupledIO
- Think about the ADT design for Command such that we can actually describe things we care about
    - e.g. mapping on a Command should lazily materialize a Chisel circuit into Calyx
    - e.g. use of ‘when’ inside a flatMap closure should actually map to a conditional branch in the control flow
    - e.g. recursion within flatMaps for a command should be handled by catching them and preventing infinite recursion inside the interpreter
    - e.g. when peeking a value we want to save a handle to it rather than the actual value (which is only present at circuit runtime anyways), the user will have to run ops that relate to the handle, not the value itself

```scala
for {
   p1: UInt, Handle[UInt] <- peek(a)
   p2 <- peek(b)
   _ <- poke(c, p1)
}
```

- See if Dahlia already defines a Calyx ADT in Scala: https://github.com/cucapra/dahlia

## 11/2/2022

- Calyx paper (HLS for Command monad)
    - How to handle synchronization across threads (e.g. channel based communication)?
    - https://docs.calyxir.org/lang/sync.html
- FIFO testbench debug
- Modify interpreter so that type is restricted to `R <: Data`, then emit Calyx IR from a Command by unrolling it (group together peeks and pokes as groups, steps are control ‘seq’ constructs). We will need For/While/If primitives in our ADT too at some point
- TODO: synthesizable Decoupled driver

## 10/26/2022

- Interpreter example can be fixed with some manual type coercion (still safe)
- Later: debug the Chisel fifo test
- We discussed the difficulty of writing tests in cocotb wrt events around the rising clock edge (common hack is to just do everything on the falling clock edge to avoid ambiguity)
- TODO Vighnesh: share gcd_tb.py for an alternative (which is still ugly)
- Synthesizable Decoupled driver
    - IO (see below)
    - TODO Vighnesh: create a strawman example of this in our old VIP repo (or in simcommand)

```scala
class Example extends Module {
   val io = IO()
   val q = Queue()
   val driver = SynthDriver()
   driver.queueIf <> q.enq
   driver.control <> io
}
test(Example()) c => drive transactions through the synthdriver IO (minimize interaction with the DUT)
baseline test(Queue()) c => drive transactions using chiseltest decoupled driver (which will definitely be slower)
```

## 10/19/2022

- FIFO in Chisel hacking (https://github.com/bdngo/chisel-fifo)
- TODO for Vighnesh: fix the interpreter example
- Work on a cocotb testbench for the fifo
- Attempt the decoupled driver for synthesis

## 10/13/2022

- Continue to work through the chisel-bootcamp Ch. 3
- Take your 151 FIFO, implement it in Chisel, and test using chiseltest
    - Also try cocotb on your original Verilog
- Command monad hacking as we discussed
    - Implement from this basis: https://vighneshiyer.github.io/#/2/7/1
    - Then create a stack overflow on purpose
    - Can you modify the interpreter to avoid overflowing the stack? Think about maintaining a stack of Commands (in your interpreter) and using an imperative implementation vs the recursive one in the presentation
- Write a DecoupledDriver for synthesis (using Chisel)
    - `class Tx[T <: Data](gen: T) extends Bundle { val data = gen }`
    - `class DecoupledDriver extends Module {`
        - `val io = {t = Input(Vec(Tx, 10)), readytoacceptmore = Output(bool)}`
    ◦ There is an industry spec for this (kind of)
        ▪ https://www.accellera.org/images/downloads/standards/sce-mi/SCE-MI_v22-140120-final.pdf
        ▪ SCE-MI: standard for describing synthesizable ‘transactors’ that we can interact with from a testbench (instead of implementing the per-interface peek/poke logic from the testbench, it is embedded in the RTL)

## 10/5/2022

- chisel-bootcamp, strawman prototype of SimCommand (then look at the code)
- Ultimate goal is to get an understanding of how the Command monad works and understand how we can use it to describe synthesizable VIPs
- Interim goal: write a fully synthesizable DecoupledIO VIP, connect it with the DecoupledGCD and elaborate the whole thing using chiseltest, then write a chiseltest testbench that interacts with the VIP by sending *transactions* to the synthesized VIP
    - Also measure the performance improvement over implementing the VIP in chiseltest with fork/join
- Once you feel somewhat comfortable, try to write your own circuit in the chisel-template (https://github.com/freechipsproject/chisel-template) and write a test for it (you can try porting a 151 lab's circuit e.g. fifo)
- SimCommand is a good jumping off point for HLS (https://github.com/vighneshiyer/simcommand)
    - See my presentation (https://vighneshiyer.github.io) and try to create your own simple recursive interpreter for your own Command monad. You will gain understanding of what's actually going on.
    - Try to use cocotb for writing a simple test (https://docs.cocotb.org/en/stable/coroutines.html), again maybe use a 151 circuit as the DUT
    - You may want to read the Calyx paper (https://calyxir.org/) to understand what an HLS IR looks like
- Our initial goal will be to define a chiseltest interface that enables passing transaction-level information to the underlying simulator with hand written RTL-based VIPs. Later we will use SimCommand and a custom interpreter with HLS to generate those VIPs.
