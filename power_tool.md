# Power Tool

## Grand Vision

We want to build an open-source, clean, from scratch "PowerPro-like tool" that supports fine-grained incrementalism, semantics-aware synthesis, and has a very low latency (seconds to minutes to evaluate an RTL change).
To do this, we need a synthesis tool.
To design a synthesis tool in a novel way that can trade-off QoR and runtime at a fine-granularity, we want to leverage egraphs somehow.
To leverage egraphs effectively and aggressively cache/dedup circuit elements we need a 'content-addressed' IR.
Then we also need a bridge from some frontend or another IR into the target IR for our tool.
We also need a gate-level simulation tool to propagate RTL-level activity to a gate-level netlist, taking into account clock periods, signal transitions around clock edges, and do this in a highly parallel manner which can strongly scale to many cores.

But, all of this is a lot of work, and while necessary in the long-run, isn't essential to get a prototype working.
The prototype is supposed to be a very fast RTL -> power/area/timing estimation tool that leverages existing tools rather than recreating them.
It might not support incrementalism, caching, or any fancy features, but should be sufficient to compare against commercial tools.
And, it is a starting point to prototype trading off runtime for accuracy, which is a curve we can't explore properly using commercial tools.

## Prior Work

The problem: I have RTL and a PDK. I have a RTL waveform. I want to know how much power is consumed by every bit of my circuit, and I want to know it *fast*.

### The Big 4

Every big EDA tool vendor (Cadence, Synopsys, Siemens, Ansys) offers a variant of this kind of tool.

#### Cadence Joules / Voltus

I wrote a long explanation of Joules as given to us by Cadence engineers [on my blog](https://vighneshiyer.com/conference_reviews/dac-2022/) (search for 'joules').
The main selling point is that Joules performs synthesis using the *same synthesis engine as Genus*, so you can trust its results.
Furthermore, since it uses a relatively detailed synthesis pass, it is able to give accurate what-if analysis results when evaluating clock gating opportunities, among other things (accurate CTS estimates, placement-aware routing estimates).

The runtime of Joules is ~20-60 minutes for a small Chipyard Rocket design, depending on CPU load and timing constraints.
It is often hard to check if Joules is set up correctly for the PDK under consideration - it can give very weird numbers sometimes (incorrect proportion of leakage vs switching vs internal power).

Joules also appears to handle RTL toggle propagation to gate-level in an odd and non-transparent way.
Changing the timebase or the clock period of the *RTL simulation*, and running that waveform through Joules, will give different power numbers.
It is also not clear if Joules is doing RTL -> GL activity propagation using the maximum parallelism possible and oftentimes the assertion % of nets will be a lot less than 100%, which is odd since it knows the entire synthesis database.

**In short**: runtime is high, accuracy is high (relative to post-PnR Voltus), setup is difficult, and can't be used rapidly in the RTL design loop.

#### Siemens PowerPro

I have some [DAC slides on PowerPro from DAC 2022](https://vighneshiyer.com/conference_reviews/dac-2022/) also on my blog.

  - Used by Apple and ARM, RTL designers seem to enjoy it reasonably well
  - Used primarily to evaluate point changes in RTL, what-if analysis wrt clock gating
  - Appears to have a lightweight synthesis engine

#### Ansys PowerArtist

No experience, although positive experiences reported by others in industry.

#### Synopsys PrimeTime PX / PrimePower



### Silimate

- look at their github
- obvious what they are doing
- quite stubborn of letting me know what they do and they have garbage marketing that obscures the point of their startup
- yosys + opensta = "faster than powerpro" "PPA" estimates

### iEDA

- iPW

### OpenROAD

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

Technology agnostic variant just for relative netlist changes and power impacts

- CTS estimation, cell sizing estimation via transformer (see NVIDIA prior work), basically try to shortcircuit synthesis to the maximum degree possible
- might be difficult to shortcircuit boolean optimization
- look into lgraph for incremental boolean opt and cell mapping (sizing maybe not considered)
- try to do naive activity propagation initially (see GRANNITE) - this is easier than trying to use a vcd, try to use a saif first

1. First is leakage power comparison vs Genus/Joules using a Genus GL netlist (NOT yosys)
2. Register dynamic switching power comparison vs Genus/Joules
3. yosys vs Genus comparison (wrt cell mapping, sizing, etc.)

saif before diving into vcds
try to use rtl level before diving into gate level
