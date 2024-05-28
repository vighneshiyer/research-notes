# RTML Bringup

## Strategy

- [ ] Select FPGA [d:5/23/2024]

### FPGA Devboard

- Things to consider: cost, FPGA type, host interface, DRAM, connector + voltages, form factor
- [Xilinx VCU118](https://www.xilinx.com/products/boards-and-kits/vcu118.html)
  - $9k
  - Virtex Ultrascale+
  - USB-UART / PCIe with riser
  - 2x 2.5 Gib DDR4
  - FMC-HPC (ASP-134486-01), VADJ = 1.8V by default, can go down to 1.2V if FMC EEPROM is programmed or if system controller is overridden (intricate, but doable)
  - Large PCIe card
- [Opal Kelly XEM7350](https://opalkelly.com/products/xem7350/)
  - $1.5k
  - Kintex 7
  - USB3 with special RTL + drivers with 340 MiB/s bandwidth
  - 512 MiB DDR3
  - FMC-HPC (-K160T part) (ASP-134486-01) (mates with ASP-134488-01), VADJ = 1.2V
- Digilent Nexys Video

#### Mechanical Sketch

- https://docs.opalkelly.com/wp-content/uploads/2022/05/XEM7350-MechanicalDrawing.pdf
  - 4x standoffs for the FPGA devboard that exceed 6.5mm in height, 2.70mm hole diameter (use M2 standoff kit, M2 has clearance hole diameter of 2.6mm)
  - 2x standoffs for the chip motherboard that are likely around 18.5mm (will use same hole diameter)

### Connector

- Bump map and connector pin assignment
- Connector
  - FMC EEPROM?

### Socket

- Bump pitch, escape routing evaluation against manufacturer DRC, via in pad necessary?
- Decap
- Which signals to route and which ones to leave floating (chip DDR outputs) / ground (chip DDR inputs)

### Power and Clocking

- Power supplies
  - IO (from FMC side) vs core (from benchtop supply) VDD
  - Isolate VDDs on split power plane
  - Short all grounds together, uniform flood plane
- Clocking

### Off-chip Interfaces

- JTAG
  - Directly place the JTAG header on the motherboard, DO NOT daisy chain chip JTAG to FMC JTAG and DO NOT pass JTAG signals through FMC
  - Use this JTAG debugger: https://www.olimex.com/Products/ARM/JTAG/ARM-USB-OCD-HL/ (**-HL variant** works down to 0.65V)
    - Digikey (https://www.digikey.com/en/products/detail/olimex-ltd/ARM-USB-OCD-HL/21662466) ($58.79)
  - Use a USB optoisolator if we're scared
    - https://www.olimex.com/Products/USB-Modules/USB-ISO/
  - Standard 20-pin ARM JTAG header (https://developer.arm.com/documentation/101416/0100/Hardware-Description/Target-Interfaces/ARM-Standard-JTAG)
    - 2 rows × 10 pins at 0.1" step
    - This will do: https://www.digikey.com/en/products/detail/sullins-connector-solutions/SBH11-PBPC-D10-ST-BK/1990065
    - Another Molex alternative that's much more expensive for unknown reason: https://www.digikey.com/en/products/detail/molex/0702462002/760185
- DDR
- UART
- TSI / Serial TL

### Debug

- Chip reset
  - LED with driver FET (make sure V_{gs,max} << 1.2V) + header pin
- Clocks
- FMC powergood
  - LED with driver FET
- Serial TL
  - Everything routed out to header pins with 0Ohm resistors

## PCB CAD

### Considerations

- Plan is to use JLCPCB with 4 layer 2mm board, 2U ENIG, epoxy filled vias (if via-in-pad is required), 0.15mm min vias, uncontrolled impedance
  - Ideally I can relax the min via diameter, but it may not be possible with the fine BGA pitch. Via-in-pad might be necessary to make 4 layer routing doable. Move to 6 layer if absolutely required.
  - Plan is to also do assembly with JLCPCB and get just fully assembled boards. Might need to special order FMC terminals, but otherwise OK
- https://jlcpcb.com/capabilities/pcb-capabilities
  - min 0.25mm BGA pads - OK
  - hole to hole clearance 0.5mm - OK
  - 0.09mm min trace width, 0.09 min trace spacing - seems sufficient for BGA escape
  - Everything looks good enough, no need for fancy board house
- https://jlcpcb.com/help/article/243-BGA-Design-Guidelines---PCB-Layout-Recommendations-for-BGA-packages
  - 0.10mm min trace to BGA pad spacing
  - Everything still seems good enough
