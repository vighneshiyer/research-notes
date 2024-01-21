# Notes on Chipyard RTL Sim Perf Regression

- huffbench using tidalsim --golden-sim on latest Chipyard = 1400 seconds
- huffbench on old Chipyard (mid-October i think) = 630 seconds
    - 2x PERFORMANCE REGRESSION!
- On latest Chipyard, performance with VCS
    - hello using default testharness = 6.420
    - hello using injection testharness = 6.380 (basically no difference)
    - dhrystone using default testharness = 89.230 seconds
    - dhrystone using injection testharness = 89.420 (basically no difference)
    - hello with verilator = 13.08 seconds (36066 cycles)
    - 1ns clock in TestDriver, 2ns clock in DigitalTop
- So the problem is due to something in Chipyard, not my custom harness
- Perhaps I should look at CI runtime first and see if there is a clear explosion after some commit
- On `e620` (e6203bb25c055e9135ee0371d95accb88108b0be) (October 5, 2023)
  - hello.riscv (VCS): 3.49 seconds
  - hello.riscv (Verilator): 8.19 seconds (36396 cycles)
  - OK, we can already see the problem - this is 2x faster, now we must bisect
  - 1ns clock in TestDriver, 2ns clock in DigitalTop

I've noticed a performance regression for RTL simulations in Chipyard from early October to today. Running `hello` (with fast loadmem) on Chipyard e6203bb (Oct 5 2023) takes 3.49/8.19 seconds (VCS/Verilator) and on latest Chipyard takes 6.42/13.1 (VCS/Verilator) seconds. I notice the same trend of > 2x performance degradation in VCS simulations for longer workloads too. Before I try to bisect, does anyone have an idea?

- We will use `hello` (with `LOADMEM=1`) as the benchmark running on default VCS
  - Baseline (Oct 23) (e6203bb) = 3.5-ish seconds
  - Current (Jan 24) = 6.6-ish seconds (2x degradation)
  - f9b5bee98 (Dec 16) = 3.5-ish seconds (no degradation)
  - c9fa23edf (Jan 8) = 3.8-ish seconds (there is a slight degradation that seems consistent) (but i'll treat this commit as still 'good', since there should be a much more significant degradation coming)
  - 8073ecb1 (Jan 11) = !!! this commit doesn't even compile! How did CI ever pass?
  ```text
[error] /scratch/vighneshiyer/chipyard-bisect2/generators/chipyard/src/main/scala/clocking/CanHaveClockTap.scala:16:13: not found: value SubsystemDriveAsyncClockGroupsKey
[error]   require(p(SubsystemDriveAsyncClockGroupsKey).isEmpty, "Subsystem asyncClockGroups must be undriven")
[error]             ^
[error] /scratch/vighneshiyer/chipyard-bisect2/generators/chipyard/src/main/scala/clocking/CanHaveClockTap.scala:19:33: not found: value asyncClockGroupsNode
[error]     clockTap := ClockGroup() := asyncClockGroupsNode
[error]                                 ^
  ```
  - 45d74f6db (Jan 11) = !!! this commit also doesn't compile! - same issue as above
    - When trying to comment out those lines, and trying to compile again, I get diplomacy exception
  ```text
Caused by: java.lang.IllegalArgumentException: requirement failed: Diplomacy has detected a problem with your graph:
At the following node, the number of inward ports should equal the number of produced inward parameters.
sink system.clockTap node:
parents: system/chiptop0
locator:  (generators/chipyard/src/main/scala/ChipTop.scala:27:35)

0 outward ports connected: []
0 inward ports connected: []

Upstreamed outward parameters: []
Produced inward parameters: [ClockSinkParameters(0.0,5.0,10000.0,200.0,None,Some(clock_tap))]
    ```
  - f84b2428 (Jan 11)
    - This is definetely where things go wrong, but the past 2 commits don't compile so I can't get numbers for them
    - Yes, we see 6.9 seconds for simulation

- CI numbers
  - Jan 5 (46585a5) chipyard-rocket-run-tests = 13 minutes
  - Jan 9 (5bc9aea) = 14 minutes
  - Jan 11 (8073ecb1) (1697 clock_tap) = 14 minutes
  - Jan 11 (f84b2428) (Fix CanHaveClockTap) = 22 minutes
  - There is one commit in the middle (merge commit), but there is no CI for it - let me try that manually
