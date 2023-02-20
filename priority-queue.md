# Priority Queue Chisel Library

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
    - See the “Status Flags” section
- Priority Queue in Chisel
    - https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9730477&tag=1
    - TODO Vighnesh: attach more links here + IO spec + our initial prototype
    - https://github.com/joey0320/chisel-priorityqueue/blob/for-huffman/src/main/scala/shift_register_pq.scala
    - https://github.com/joey0320/chisel-priorityqueue
    - https://ieeexplore.ieee.org/document/895938
    - http://fmdb.cs.ucla.edu/Treports/140013.pdf
