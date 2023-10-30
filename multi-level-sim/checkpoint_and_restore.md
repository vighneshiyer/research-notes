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

#### 10/23/23

- [x] compare the two waveforms
    - Look at s2_pc, clock, reset, icache fetch activity

- Reference waveform:
    - Processor comes up with PC = 0x10040, s2_pc frozen until icache resp comes back, moves until 0x1_006c (wfi), upon ipi jumps to 0x1_0000 (_start), then fetches from icache, goes until 0x1_00bc, then jumps to 0x8000_0000, then after long latency of DRAM load, begins execution and terminates eventually
- Injected waveform:
    - Processor comes up with PC = 0x10044, since we skip loading position of `_start` mtvec is loaded with the random contents of `a0` as expected, execution continues until 0x1_006c (wfi), upon ipi jumps to garbage mtvec address (0xc9d4_4752), icache doesn't even set a_valid to high, then simulation stalls
    - Wait, this actually looks good lol - we see the IPI, processor jumps to garbage address, move onto next trial

- Next steps:
    - Remove the MSIP interrupt stuff from fesvr
    - Try to jump immediately to 0x8000_0000 with PC setting
    - There might be more fishy-ness with having to set some CSRs that the bootrom does

- `make -j16 run-binary-debug LOADMEM=1 STATE_INJECT=1 BINARY=$RISCV/riscv64-unknown-elf/share/riscv-tests/isa/rv64ui-p-simple`
    - Also set SIM_FLAGS to +no_hart0_msip when STATE_INJECT=1, let's see
    - Cool, this seems to work, -p-simple and -p-add run to completion when skipping the bootrom altogether
- this doesn't happen with bash using native system path, lol, so need to install my stuff from source now instead of using junest

#### 10/25/23

- Now trying to get testchip_dtm loadarch stuff pulled out. The verilator command is:

```bash
verilator --main --timing --cc --exe -CFLAGS " -O3 -std=c++17 -I/scratch/vighneshiyer/chipyard/.conda-env/riscv-tools/include -I/scratch/vighneshiyer/chipyard/tools/DRAMSim2 -I/scratch/vighneshiyer/chipyard/sims/verilator/generated-src/chipyard.harness.TestHarness.RocketConfig/gen-collateral   -DVERILATOR -include /scratch/vighneshiyer/chipyard/sims/verilator/generated-src/chipyard.harness.TestHarness.RocketConfig/chipyard.harness.TestHarness.RocketConfig.plusArgs" -LDFLAGS " -L/scratch/vighneshiyer/chipyard/.conda-env/riscv-tools/lib -Wl,-rpath,/scratch/vighneshiyer/chipyard/.conda-env/riscv-tools/lib -L/scratch/vighneshiyer/chipyard/sims/verilator -L/scratch/vighneshiyer/chipyard/tools/DRAMSim2 -lriscv -lfesvr -ldramsim "   --threads 1 --threads-dpi all -O3 --x-assign fast --x-initial fast --output-split 10000 --output-split-cfuncs 100 --assert -Wno-fatal --timescale 1ns/1ps --max-num-width 1048576 +define+CLOCK_PERIOD=1.0 +define+RESET_DELAY=777.7 +define+PRINTF_COND=TestDriver.printf_cond +define+STOP_COND=!TestDriver.reset +define+MODEL=TestHarness +define+RANDOMIZE_MEM_INIT +define+RANDOMIZE_REG_INIT +define+RANDOMIZE_GARBAGE_ASSIGN +define+RANDOMIZE_INVALID_ASSIGN +define+VERILATOR --top-module TestDriver --vpi -f /scratch/vighneshiyer/chipyard/sims/verilator/generated-src/chipyard.harness.TestHarness.RocketConfig/sim_files.common.f +define+DEBUG  -o /scratch/vighneshiyer/chipyard/sims/verilator/simulator-inject-chipyard.harness-RocketConfig-debug  --trace -Mdir /scratch/vighneshiyer/chipyard/sims/verilator/generated-src/chipyard.harness.TestHarness.RocketConfig/chipyard.harness.TestHarness.RocketConfig.debug -CFLAGS "-include /scratch/vighneshiyer/chipyard/sims/verilator/generated-src/chipyard.harness.TestHarness.RocketConfig/chipyard.harness.TestHarness.RocketConfig.debug/VTestDriver.h"
```

```cpp
#include "loadarch.h"

int main(int argc, char* argv[]) {
  assert(argc == 2);
  std::string loadarch_file(argv[1]);
  auto l = loadarch_from_file(loadarch_file);
  printf("%d\n", l.size());
  printf("%lx\n", l[0].pc);
  for (int i = 0; i < 32; ++i) {
    printf("%lx\n", l[0].XPR[i]);
  }
}
```

```bash
g++ --std=c++17 \
-I/scratch/vighneshiyer/chipyard/.conda-env/riscv-tools/include \
-L/scratch/vighneshiyer/chipyard/.conda-env/riscv-tools/lib -Wl,-rpath,/scratch/vighneshiyer/chipyard/.conda-env/riscv-tools/lib \
-lriscv -lfesvr \
loadarch.h loadarch_main.cc -o loadarch_main
```

- K looks good, next add DPI hook

#### 10/26/2023

- Regular DTM loadarch still works fine
- Trying to write struct from C-land, which seems fine too
- Oh VCS doesn't recognize the DPI function, probably because it is in a .h file :(
    - Time to refactor this a bit again

#### 10/27/2023

```text
Error-[SV-NYI-RSUDD] Unsupported SystemVerilog feature
/scratch/vighneshiyer/chipyard/sims/vcs/generated-src/chipyard.harness.TestHarness.RocketConfig/gen-collateral/TestDriver-inject.v, 170
"force TestDriver.testHarness.chiptop0.system.clint.timecmp_0 = loadarch_state.mtimecmp;"
  Argument: loadarch_state.mtimecmp
  Reference to a member of a structure containing dynamic data and/or used in
  dynamic arrays is not yet supported in non-procedural context.


Error-[DTINPCIL] Dynamic type in non-procedural context
/scratch/vighneshiyer/chipyard/sims/vcs/generated-src/chipyard.harness.TestHarness.RocketConfig/gen-collateral/TestDriver-inject.v, 182
"force TestDriver.testHarness.chiptop0.system.tile_prci_domain.tile_reset_domain_tile.fpuOpt.regfile_ext.Memory[i] = loadarch_state.FPR[i];"
  Argument: i
  Automatic variable may not be used in non-procedural context.
```

- So looks like the struct fields must be 'static' somehow
- OK this is resolved, reloading -simple after 10 instructions in spike seems to work!
- However, hello hangs :(
- Need to investigate reload of mstatus more carefully (there are upper order bits set from the spike checkpoint that aren't reloaded in RTL sim)
- But first check other riscv ISA tests - need to check if XPRs are being loaded properly
- Also need to try simple after a few more instructions (not just 10)
    - Also need to check that the command line flags are right, no need to specify LOADARCH

#### 10/28/2023

- First debug rv64ui-p-simple and all its checkpoint variations
- Hmm maybe the problem is pmp related?
    - rv64ui-p-simple expects csrw to a pmp register to work, but with --pmpregions=0, spike throws illegal instruction and traps (but exits with tohost=0?, which is odd)

#### 10/29/2023

- OK so it looks like the PMP thing is definetely the culprit for the rv64ui-p-simple mismatch
    - In pure RTL sim, it passes clean
    - In pure spike sim, it also passes clean (but very oddly imo)
- The solution might just be to create our own spike top level (for easy checkpointing and fine control over instruction steps / commit log printing) and also modify spike if needed to expose all the arch state we want to extract
    - And also create a way to restore checkpoints in spike and continue execution (which right now, we can't do)
- [x] Figure out why spike doesn't exit with bad tohost code for rv64ui-p-simple when run with `--pmpregions=0`
    - Indeed, when --pmpregions=0, spike takes an illegal inst exception, however, since the INIT_PMP block sets mtvec to the end of the block, taking the trap just skips execution of the PMP block and everything else goes smoothly
    - OK, so when this snapshot is loaded in the RTL, then it should also proceed just fine and have nonzero exit code!
    - Also, skipping the PMP block seems unrelated to the failure...
- [x] Inspect the loadarch file after 50 insts committed and correlate the RTL commit log to the spike commit log with `--pmpregions=0`
    - At 50 insts: PC = 0x0000000080000114
    - The first point of divergence between DMI RTL commit log and spike commit log is [0x8000_0178 (fence)](https://github.com/riscv/riscv-test-env/blob/4fabfb4e0d3eacc1dc791da70e342e4b68ea7e46/p/riscv_test.h#L248)
        - The fence causes a trap to the `trap_vector` and then things go haywire in RTL
    - No clue why a fence would cause a trap - maybe this is PMP related?
- [x] Compare DMI checkpointed log with vanilla DMI-based bringup log of the full program
    - So the vanilla DMI log seems fine and matches spike - the fence doesn't trigger a trap, just falls thru and traps only on ecall as expected
    - So I suspect PMP issue
- [x] Figure out what the PMP registers should be and force them
    - pmpaddr0 = 0x001fffffffffffff
    - pmpcfg0 = 0x000000000000001f
    - After forcing these, rv64ui-p-simple checkpoint loaded into RTL sim with force works! For 50 and 60 instruction checkpoints.
    - `hello.riscv` still hangs though :(
    - More testing with asm tests and benchmarks is required

#### 10/30/2023

- [x] Run rv64ui-p-add on several instruction breakpoints
    - Works on 10,50,1000,2000 :(, no interesting findings
- Fishy
  - Top bits of mstatus
  - Setting mstatus_fs, xs, vs preemptively
  - FCSR flag bits not being set
  - PMP state
  - PRV?
  - Check if any arch state bits are set to 1 that aren't forced
