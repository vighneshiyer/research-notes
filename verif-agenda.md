# High Level Verification Agenda

## Sketches

- Writing testbenches from Scala (this is what we want to demonstrate is not only possible, but better than the status quo in every way)
    - Performant
    - Ergonomic
    - Correct (good reference implementations)
- chiseltest (the current implementation)
    - Slow (very slow esp when you use fork/join)
    - There are no standardized VIPs, or transaction hierarchies
    - No golden models, no properties, no assertions, no verification of existing Chisel IP (no references for others to build on)
- Everyone here is working on addressing each of these concerns:
    - Shawn: standardize transactions (across bus types) and have reference VIPs and their implementations
    - Bryan: moving software components of testbenches (VIPs) to RTL (using HLS) - allows use of FPGA accelerated simulation
    - Oliver: If you want to write testbenches you have a few options (Verilog, SystemVerilog + UVM, Verilator + C++, cocotb + Python, chiseltest + Scala + SimCommand). We want to make chiseltest + Scala + SimCommand performance competitive with SOTA commercial Verilog simulators and Verilog testbenches.
