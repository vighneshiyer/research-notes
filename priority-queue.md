# Priority Queue Chisel Library

- Repo: https://github.com/william-lyh/chisel-priorityqueue

## 3/7/2023

- In the future, other PQ architectures probably will use SRAMs internally
    - SRAMs in Chisel are SyncReadMem (https://www.chisel-lang.org/api/3.3.3/chisel3/SyncReadMem.html)
    - In this case, we are using registers since we need to shift a bunch of entries on the same cycle
- Next things:
    - Make some of the IOs boolean
        - wrEn, rdEn Bool()
    - Add priority out as an output along with dOut
    - Make the PQ type generic on the data type
    - Later: make the priority type also generic by make it a generic type that needs a PartialOrder typeclass instance
        - https://typelevel.org/cats/api/cats/kernel/PartialOrder.html
- Formal verification of the PQ
    - Watch this lecture: https://www.youtube.com/watch?v=ssAbq5tdh8Y
    - Try out some examples from the lecture in your own environment (in the priorityqueue repo)
    - https://github.com/agile-hw/lectures/blob/main/22-formal/lec22-formal.ipynb
- Evaluate other architectures
    - Check the priority queue paper and sketch out the algorithm for another implementation (based on binary trees / tries / etc.)
    - There are also some non-exact architectures that don't produce data in the priority order, but are very close (and are more area efficient). Do a literature review and come back with a few papers and ideas.

## 2/20/2023

- https://github.com/EECS150/fpga_labs_sp22/blob/master/lab6/spec/spec.md
    - FIFO exercise for implementation in Verilog
    - Understand the testbench: https://github.com/EECS150/fpga_labs_sp22/blob/master/lab6/sim/fifo_tb.v

```scala
class PriQueue(nEntries: Int) extends Module {
    val io = IO(new Bundle {
        val enqValid = Input(Bool()) // data, priority, ready, valid
        val enqReady = Output(Bool())
        val enqData = Input(UInt(8.W))
        val enqPri = Input(UInt(2.W))

        val deqValid = Output(Bool())
        val deqData = Output(UInt(8.W))
        val deqReady = Input(Bool())
        val deqPri = Output(UInt(2.W))
    })
    // accept up to nEntries of input data
    // always drive the highest priority piece of data first
    // pieces of the data with the same priority should be driven in FIFO order
}
```

- yosys (for "schematic" style visualization of Verilog) (or use Vivado elaboration - see `make elaborate` in the 151 labs)
    - `read_verilog <verilog file name>`
    - `hierarchy`
    - `proc`
    - `show`
- https://github.com/freechipsproject/diagrammer

```verilog
module ex();
    wire a;
    assign a = (b == c) ? d : e;

    reg a;
    always @(*) begin
        if (b == c)
            a = d;
    end

    always @(posedge clk) begin
        if (b == c)
            a <= d;
    end

    assign fake = b == c;
    ff name(.d(d), .q(a), .en(fake), .clk(clk))
endmodule
```

## 11/10/2022

- FIFO and testbench look good
    - For the FIFO, can you avoid tracking the number of elements in the FIFO altogether? Can you reduce thes number of bits of state in the FIFO?
    - Hint: see the Wikipedia page for FIFO
    - https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)
    - See the ???Status Flags??? section
- Priority Queue in Chisel
    - https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9730477&tag=1
    - TODO Vighnesh: attach more links here + IO spec + our initial prototype
    - https://github.com/joey0320/chisel-priorityqueue/blob/for-huffman/src/main/scala/shift_register_pq.scala
    - https://github.com/joey0320/chisel-priorityqueue
    - https://ieeexplore.ieee.org/document/895938
    - http://fmdb.cs.ucla.edu/Treports/140013.pdf
