# UCIe VIP Design

- UCIe digital repo: https://github.com/ucb-ucie/uciedigital/
- Initial goal is to create a FDI interface VIP for the UCIe D2D + PHY stack so we can test communication between 2 UCIe attached devices
    - Along the way, we will discover additional signals that need to flow from the PHY/sideband (in addition to RDI) to the D2D adapter

## 10/4/2023

### Getting the Testbench Template Up

1. **Me**: Write and elaborate a chisel module that has the Fdi Bundle as one of the input IOs (representing the D2D adapter)
    - might need to DontTouch the Fdi interface so Chisel/firrtl doesn't optimize it out
    - now you have a Verilog DUT with Fdi interface
1. Write some Scala to turn the Fdi Bundle object into a systemverilog interface and emit that alongside elaborating the D2D dummy module
    - you can do (new Fdi()).getElements: Seq[(String, Data)]
    - then use this to generate a systemverilog interface
1. Write a SV top that wraps the dummy D2D module + attaches an interface instance to the D2D module's ports
    - you can also generate this file from Scala alongside elaborating the D2D dummy module
    - now you have a SystemVerilog DUT wrapper w/ an interface that you testbench can use
1. Write a simple SV testbench
    - instantiate the DUT with the interface
    - just drive some signals at the top-level for fun
1. Simulate this with VCS / (try Verilator too)
    - pipeclean the flow
    - make sure all the required commandline flags are present
    - make sure we can generate a waveform, even though the module is doing nothing

### Getting a Basic VIP Up

1. Write a SV VIP that speaks the lp interface
    - define an interface that contains just the lp signals
    - define a simple transaction type that just places some data on the lp_data bus using the Decoupled3 protocol
        - Define a systemverilog class that has some fields (e.g. data)
        - `class FDITransaction begin logic [] data; /* add CRV constraint blocks too */ end`
    - write the implementation for the driver that actually does the cycle-by-cycle driving of the transaction on the lp_data bus (using the Decoupled3 protocol)
- Later: script up splitting the lp interface from the pl interface in Chisel

- Chisel -> D2D adapter RTL in Verilog
    - Generated string: SystemVerliog interface for the D2D adapter FDI interface
    - Generated string: SystemVerilog DUT wrapper for the RTL and the SystemVerilog interface

- Dump the code here: https://github.com/ucb-ucie/uciedigital/tree/main/src/main/scala/d2dadapter initially
    - Do this on a branch first, don't push to master yet
