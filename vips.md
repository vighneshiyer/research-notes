# VIP Development

## 1/31/2023

- Past week, reviewed the TL spec and the TL-UL protocol
    - Very basic, 2 channels, A channel goes from master to slave, D channel goes from slave to master
    - Memory read = Get (A) -> AccessAckData (D)
    - Memory write = PutFullData(A) | PutPartialData(A) -> AccessAck(D)
- Earlier implementation of a TileLink VIP and fuzzer: https://github.com/TsaiAnson/verif

### Vighnesh's TODOs

- Republish recent versions of rocket-chip, rocket-dsptools, and dsptools itself
    - Do it under my domain
- Construct a basic TLRAM example for TL VIP work
- Draft a project proposal for you that's based on building a hardware datatype library for Scala

## 1/25/2023

- tapeout, physics 7b + 137a, ee 117, 105
- SURF, ...

- Read through the TileLink spec (only from the start to Ch. 6 - TL-UL) (https://static.dev.sifive.com/docs/tilelink/tilelink-spec-1.7-draft.pdf)
    - Memory interaction primitives: Read, Write, AtomicCAS (compare and swap), ...

    ```scala
    sealed trait MemPrim {
        def toTLPrim(): TLPrim = {
            ???
        }
    }
    case class Read(addr: BigInt) extends MemPrim
    case class ReadResponse(data: Seq[Byte]) extends MemPrim
    case class Write(addr: BigInt, data: Seq[Byte]) extends MemPrim

    sealed abstract class TLPrim(dataWidth: Int)
    case class Get(data: BigInt, addr: BigInt, corrupt, ...) extends TLPrim
    // ... all the various tilelink message types
    ```

    - Develop a data type that models these memory interaction primitives
    - Write a function that can convert between a Read/Write and the TileLink transaction primitives (TileLink messages)
    - Extra: Write function to convert between same memory primitives and the AXI4-Lite transaction primitives
- Develop a VIP that can drive TileLink transactions into the DUT (some TileLink module - maybe a RAM, width converters, buffers, ...)
- TODO: produce a repo that elaborates a TileLink module out-of-context so it can be tested using chiseltest

## 11/30/2022

- TODO Vighnesh: standalone AXI module implementation for you to test your AXI VIP on

```scala
sealed trait MemTx
case class Read(address: Long, numBytes: Int) extends MemTx
case class Write(address: Long, data: BigInt, numBytes: Int) extends MemTx

sealed trait AXITx
case class ARMessage(arid: Int, arlen: Int, arSize: Int, arBurst: Int, ... see other fields in spec) extends AXITx
case class AWMessage(awid: Int, awlen: Int, awsize: Int, burst: Int) extends AXITx
case class WMessage(wid: Int, wdata: BigInt, wstrb: Int, wlast: Int) extends AXITx

def convert(memTxns: Seq[MemTx]): Seq[AXITx]
```

- In reality, this conversion doesn't happen eagerly, but rather is a stateful conversion that happens iteratively as the AXI peripheral is simulated - but this is sufficient for now to get you familiar with the AXI spec
- Write the convert function for AXI4-Lite first (https://developer.arm.com/documentation/ihi0022/e/AMBA-AXI4-Lite-Interface-Specification), and then move on to the full AXI4 interface, and then even the TileLink interface

## 11/16/2022

- TODO Vighnesh: get rocket-chip dependency resolved and get function prototype for AXI driver/monitor written for Shawn to attempt implementation
    - Also need to get diplomatic AXI SRAM elaborated for testing

## 11/11/2022

- https://docs.scala-lang.org/overviews/scala-book/introduction.html
    - This is one resource for learning Scala 2
- TODO: I want to have a function that takes in MasterDrvTx and SlaveDrvTx (which you will design) and returns enq and deq interface MonTx’s.
- TODO Vighnesh: add this code inside our consolidated repo, and give you a handle on writing VIPs for AXI
    - Later: we want to write these VIPs in SimCommand instead of using raw chiseltest
    - https://github.com/vighneshiyer/simcommand
    - TODO: go through the README and my presentation (https://vighneshiyer.github.io/) and ask questions

## 11/2/2022

- How to rewrite the monitor function as something that returns an ‘iterator’ of transactions that can be lazily pulled
- TODO: write a test for the monitor (alongside the driver and receiver)
    - Place the monitor on one interface and make sure it can get MonTx
    - TODO: find a way to get the cycleStamp working again
- Read this: https://cluelogic.com/2011/07/uvm-tutorial-for-candy-lovers-overview/
- TODO Vighnesh: add rocket-chip dependency to our project
    - Provide you with a simple AXI interface Bundle that you can write a VIP for
    - Also migrate your VIP to simcommand

## 10/26/2022

- `testOnly <class name> – -z “test name”` (to only run one test in a class)
- We made everything type generic
- Randomize backpressure from the receive interface
    - When valid is high that means we have valid data to receive
    - If ready is not high, then that means the receiver is placing *backpressure* on the source of data
    - We want to randomize the duration of this *backpressure* period
    - Right now we are doing no backpressure in the general case (immediately set ready to high)
- General thing:
    - investigate how we want to encode timing parameters in transactions
    - we may want to split driven transactions and monitored transaction
    - driven txns (cyclestowaitbefore, cyclestowaitafter, bits)
    - receiver txns (???, adjustment to the backpressure strategy)
    - monitored txns (bits, cycle timestamp)
- Much later: Sequences (lists of transactions in the abstract that are given concrete timing parameters by the sequencer)
- TODO Vighnesh: port the VIP to use SimCommand

## 10/19/2022

- https://github.com/Shawn-Kong/chisel-template
- Looks good
- Next: let’s work on the parallel driver/receiver now using fork/join just to get an idea of what works
    - See: https://github.com/EECS150/fpga_labs_fa22/blob/master/lab5/sim/fifo_tb.v#L291 for an example of using fork/join in Verilog to drive a FIFO from both interfaces
    - You can do fork/join in chiseltest using a similar syntax

```scala
fork {
   drive(...)
}.fork {
   receive(...)
}.join()
```

- Try this out and make sure your test passes
- Next: make the driver/receiver/transaction type generic
    - Write VIPs using chiseltest and later SimCommand and benchmark them against each other
    - Writing bus VIPs (AXI4, TileLink)

## 10/13/2022

- Write the receiver
- https://github.com/Shawn-Kong/chisel-template
- You can use gtkwave to look at waveforms
- Use chiseltest fork/join to run the receiver and driver in parallel. Here is an example (https://github.com/ucb-bar/chiseltest/blob/main/src/test/scala/chiseltest/tests/ShiftRegisterTest.scala)
- Also look at the chiseltest version of the driver and receiver when you’re done: https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/DecoupledDriver.scala
- Next: make the driver/receiver/transaction type generic

## 10/5/2022

- write a testbench for the Queue using your own VIP
    - chiseltest (DecoupledDriver, DecoupledMonitor)
    - Use the chisel-template (https://github.com/freechipsproject/chisel-template)
    - Recall our discussion on transaction-level verification and how the VIPs translate between transactions <-> wire/cycle-level interactions with the DUT
    - think about and prototype a transaction-level testbench and golden model for the Queue in chisel-template
        - how should we model concurrent port transactions?
        - can we split the dequeuing and enqueueing functions so that they use different types of transactions?
        - See an old version for inspiration: https://github.com/TsaiAnson/verif/blob/master/core/src/DecoupledVIP.scala
    - Investigate the SystemC modeling language on your time - https://www.learnsystemc.com/basic/hello_world
    - Review the AXI tutorial: https://developer.arm.com/documentation/102202/0300/Overview
