# Spike Arch Checkpoint + RTL Restore Notes

- Trying to save and restart sim of `hello` and `rv64ui-p-simple` in RTL sim seems to work fine actually
    - No segfaults as far as I can see

```text
./scripts/generate-ckpt.sh -b tests/hello.riscv  -i 1000
Generating loadarch directory hello.riscv.0x80000000.1000.loadarch
Generating state capture spike interactive commands in hello.riscv.0x80000000.1000.loadarch/cmds_tmp.txt
Capturing state at checkpoint to spikeout
Finding tohost/fromhost in elf file
Compiling memory to elf
riscv64-unknown-elf-ld: warning: cannot find entry symbol _start; not setting start address
```

```text
[I] /s/v/c/s/verilator make CONFIG=dmiRocketConfig run-binary LOADARCH=../../hello.riscv.0x80000000.1000.loadarch
Running with RISCV=/scratch/vighneshiyer/chipyard/.conda-env/riscv-tools
if [ "../../hello.riscv.0x80000000.1000.loadarch/mem.elf" != "none" ] && [ ! -f "../../hello.riscv.0x80000000.1000.loadarch/mem.elf" ]; then printf "\n\nBinary ../../hello.riscv.0x80000000.1000.loadarch/mem.elf not found\n\n"; exit 1; fi
(set -o pipefail &&  /scratch/vighneshiyer/chipyard/sims/verilator/simulator-chipyard.harness-dmiRocketConfig +permissive +dramsim +dramsim_ini_dir=/scratch/vighneshiyer/chipyard/generators/testchipip/src/main/resources/dramsim2_ini +max-cycles=10000000   +loadmem=../../hello.riscv.0x80000000.1000.loadarch/mem.elf +loadarch=../../hello.riscv.0x80000000.1000.loadarch/loadarch +verbose +permissive-off ../../hello.riscv.0x80000000.1000.loadarch/mem.elf </dev/null 2> >(spike-dasm > /scratch/vighneshiyer/chipyard/sims/verilator/output/chipyard.harness.TestHarness.dmiRocketConfig/hello.riscv.0x80000000.1000.loadarch.out) | tee /scratch/vighneshiyer/chipyard/sims/verilator/output/chipyard.harness.TestHarness.dmiRocketConfig/hello.riscv.0x80000000.1000.loadarch.log)
[UART] UART0 is here (stdin/stdout).
Hello world from core 0, a spike
loadarch attempting to load architectural state from ../../hello.riscv.0x80000000.1000.loadarch/loadarch
loadarch restoring csr 300=1e600
loadarch restoring freg 0=ffffffff00000000
loadarch restoring freg 1=ffffffff00000000
...
loadarch restoring csr 3=0
loadarch restoring csr 105=0
...
loadarch restoring reg 1=800003b4
loadarch restoring reg 2=8002af30
...
loadarch resuming harts
- /scratch/vighneshiyer/chipyard/sims/verilator/generated-src/chipyard.harness.TestHarness.dmiRocketConfig/gen-collateral/TestDriver.v:158: Verilog $finish
```

## What's Going On Under the Hood

### generate-ckpt.sh script

- Takes in: nharts, binary elf, checkpoint point (either PC or N committed insns), ISA string, output checkpoint dir
- By default, launches spike with 1 hart, PC=0x8000_0000, and immediate checkpoint capture (+ 256 MiB from 0x8000_0000 base addr for memory region, 0 pmp regions)
- Creates a spike debug command file with the following actions:
    - `until pc 0 <PC>`: Run binary until hart 0 PC = 0x8000_0000 (basically run until the bootrom is done and we have reached the start of the user binary)
    - `rs <INSNS>`: Continue running until N insns have committed
    - `dump`: Dump a raw .bin file containing memory contents to `mem.0x<addr>.bin`
    - `pc <hartid>`: Show the current PC
    - `priv <hartid>`, `reg <hartid> <csrs>`, `mtime`, `mtimecmp`, `freg <hartid> <fpr>`, `reg <hartid> <gpr>`, `vreg <hartid>`
        - Dump all architectural state by printing it to stdout
- Runs spike with the debug commands driving its execution: `spike -d --debug-cmd=$CMDS_FILE $SPIKEFLAGS $BINARY 2> $LOADARCH_FILE`
- Converts .bin memory dump to .elf with tohost/fromhost markers and .data segment at 0x8000_0000
- In practice
    - `spikecmd.sh`: `spike -l -d --debug-cmd=rv64ui-p-simple.0x80000000.10.loadarch/cmds_tmp.txt -p1 --pmpregions=0 --isa=rv64gc -m2147483648:268435456 /scratch/vighneshiyer/chipyard/.conda-env/riscv-tools/riscv64-unknown-elf/share/riscv-tests/isa/rv64ui-p-simple`
    - mem.elf goes from 0x0 to 0x1000_0408 (a bit more than 256 MiB, not sure why, but the last bit of the file contains actual data, not zeros)

### Continuing Execution in Spike (in theory)

- To do this, run spike with `mem.elf` with halted core `-H`
- And then through some magic, interact with spike via DMI to inject the arch state
    - Turns out this can only be done through OpenOCD which is too much trouble to work with
- Then resume the hart and it should run to completion
- This is what happens in RTL simulation

### DMI-Based Arch State Injection in RTL Sim

- `ChipTop` has `dmi_*` ports at the top-level - just basic req/resp with address and data
- `TestchipSimDTM` is attached to those ports
    - It contains a DPI function `debug_tick` that drives the ports defined in `testchip_dtm.cc`
- `testchip_dtm.cc` constructs `testchip_dtm_t` which extends `dtm_t` (spike) and `testchip_htif_t` (testchipip)
    - It uses VPI to read the `+loadarch` plusarg
    - Through some magic, the `testchip_dtm_t::reset()` function is called which pokes the DMI and does the arch state restoration
    - Then the harts are resumed via DMI and simulation proceeds as usual

### Fast Loadmem

- `+loadmem` - see SimDRAM.cc

## Implementation Strategy

- I'm not sure if I should modify TestDriver (in rocket-chip) directly with a single DPI function that takes the loadarch file and returns the state to be injected
- Other option is to modify TestHarness with an injected Verilog fragment which calls the same DPI function
    - afaik, the fragement must be *injected* and not within a submodule since parent references in hierarchical paths may be iffy
- After some thought, creating a new `TestDriver.v` that's swapped in with a makevar seems like the easiest option

### Hacking

#### 10/16/23

- First tried to force `s1_pc` in the frontend - well that didn't work and Rocket just jumped back to the reset vector 10040
    - This makes sense, `s1_pc` is never reset, only `s2_pc` with the hardcoded bootrom address
- Then tried to force `s2_pc` to `8000_0010`
    - My expectation is that this will start execution in the trap handler and we will then die
    - But what happened is that `s1_pc` jumps between 14 and 10 every cycle
    - And `s2_pc` is frozen at 10
    - So, a few potential issues
        - The bootrom is doing some setup to the arch state that I'm not doing
        - I need to force additional registers (unlikely since `s2_pc` is the only one being reset in the frontend that's PC specific)
        - There is something weird about jumping to the trap_vector right away, maybe just try jumping to the reset vector first
- Force `s2_pc` to `8000_0048` (start of reset_vector)
    - Same observation: `s1_pc` jumps between 4c and 48 every cycle
    - So there is some interaction with the bootrom and fesvr triggering the core to come up
    - Next: use fast loadmem to make sure fesvr doesn't need to use TSI after reset
    - https://github.com/ucb-bar/testchipip/blob/master/src/main/resources/testchipip/csrc/testchip_htif.cc#L20C2-L20C2 +no_hart0_msip seems good
- So nominally, the bootrom (https://github.com/ucb-bar/testchipip/blob/45bf0619eb040000a0efbb2902e747ad0cd17432/src/main/resources/testchipip/bootrom/bootrom.S#L4) waits for interrupt from fesvr
    - https://chipyard.readthedocs.io/en/stable/Customization/Boot-Process.html#chipyard-boot-process
    - I don't want this - the core should begin execution immediately

#### 10/17/23

- The very first thing I want to do is prove that I can set the initial PC of Rocket via force statements
- I want to show that I can start the core at 0x10044 instead of 0x10040, which should cause a garbage address to be written to mtvec so that the core goes to the garbage address upon MSIP interrupt

- [x] reference waveform using fast loadmem and rv64ui-p-simple
- [x] injection waveform using fast loadmem, rv64ui-p-simple, and custom TestDriver.v that injects PC = 0x10044
- [ ] compare the two waveforms
    - Look at s2_pc, clock, reset, icache fetch activity
