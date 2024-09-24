# Power Tool

## Grand Vision

We want to build an open-source, clean, from scratch PowerPro that supports fine-grained incrementalism, semantics-aware synthesis, and .
To do this, we need a synthesis tool.
To make a synthesis tool in a novel way, we would want to leverage egraphs somehow.
To leverage egraphs effectively and aggressively cache/dedup circuit elements

We also need a RTL simulation tool.

## Prior Work

### Commercial

- Joules
  - See DAC article
- PowerPro
  - Used by Apple and ARM, RTL designers seem to enjoy it reasonably well
  - Used primarily to evaluate point changes in RTL, what-if analysis wrt clock gating
- Silimate
  - look at their github
  - obvious what they are doing
  - quite stubborn of letting me know what they do and they have garbage marketing that obscures the point of their startup
  - yosys + opensta = "faster than powerpro" "PPA" estimates

### Academic

- iEDA
- OpenROAD
  - [IR Drop Analysis](https://openroad.readthedocs.io/en/latest/main/src/psm/README.html)
  - Nothing similar to PowerPro

## Minimum Viable Prototype

We don't want to reinvent the whole stack yet, so we should leverage tools we already have.
Let's try to understand the optimization vs fidelity tradeoff at a high level before going too deep.

1. Use yosys to perform regular synthesis, perhaps with fewer optimization steps. Perform generic gate mapping and technology mapping. Emit the netlist from yosys as blif.
  - Start with skywater130 or asap7 and switch to intel16 later.
1. Parse the [blif](https://github.com/joonho3020/blif-parser) netlist.
1. Parse the [liberty files](https://docs.rs/liberty-parse/latest/liberty_parse/) for the PDK too. For now, assume no SRAMs.
1. For every cell used in the blif netlist, find its leakage power as reported in the corresponding liberty file, add them together, and report that.

This is absolute minimum thing we should get working, and we should immediately correlate this with the output from Joules and Genus.
We can start with simple circuits (gcd, sha3, aes128).
Next step is to get a sane SRAM model working (whether that be from a fakeram generator, the actual SRAM generator/cache from the PDK, or CACTI).

Your tool should take a blif netlist, a path to a PDK's stdcell directory, and should emit a module-level report of leakage power.
