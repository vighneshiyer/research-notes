# Microarchitecture Analysis Media

My instinct is that there is a significant audience of people (people who work in semiconductors, laypeople interested in technology, uarch researchers) who are looking for highly recent microarchitecture analysis of commercial silicon (all kinds of chips intended for datacenter, desktop, mobile, embedded, and devboards).

## The Product

I would expect coverage of these kinds of chips:

I would expect coverage of these venues and relevant technology projections:

I would expect these kinds of analyses
Why isn't anyone doing a good job of this?

## Existing Players

Systematic comparison of commercial silicon running similar benchmarks (both for desktop e.g. Phoronix and datacenter e.g. stuff we will figure out)

HighYield

## Opportunities

Not merely running benchmarks, but analyzing off chip interfaces and their utilization (DDR bandwidth and characterization via mess like things) and doing top-down uarch analysis to determine how each CPU is handling different workloads. And in the case of Intel vs AMD, analyzing instruction mixes and ISA extension usage to determine how much benefit comes from arch dependent SIMD or other exotic instructions vs simply better/larger uarch structures. Comparison of number of retired of instructions and uops, IPC trace using sampling. Lots of data we could extract and analyze that others don't bother to (just report top-line numbers without analysis / reverse engineering). Evaluate impact of turning -march=native optimizations vs using an upstream generic binary (and redo the above analyses).
Same goes for power analysis, try to build a setup where we can measure core power with detailed current monitors separately from the board power. Currently, most people just report wall power, but we have the modular PSU, which should enable very precise per-pin power measurements + any on-chip power monitors we can access. Be able to analyze static/idle power in multiple sleep states and during OS idling, try to estimate the power consumed by each functional unit and uncore structures depending on activity.
