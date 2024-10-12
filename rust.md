# Rust for Baremetal RISC-V on Chipyard SoCs

## Motivation and Goals

The existing baremetal programming environment is quite bare-bones and isn't ergonomic to use.
There is no reason we can't have an allocator, clean debug APIs, and pthread-like multithreading in a baremetal environment.


Rust-based baremetal programming environment for RISC-V (specifically our Chipyard SoCs w/ htif host tethering)

- Need to port libgloss-htif to a Rust BSP
- Need to get multithreaded baremetal code working, make it easy to use 'pthreads' like APIs baremetal
- Need ability to use a baremetal allocator
- Produce very clean shrinkwrapped binaries with zero dependencies and zero libc nonsense

- Goal is to get many [no_std] rust libraries working cleanly with RISC-V and leverage them to generate a bunch of useful benchmarks we couldn't do otherwise (would require random C++ libraries running on top of an OS).
  - crossbeam no_std/alloc primitives for multithreaded baremetal programming
  - (de)compression / serde (json, bincode, flatbuffers - iffy) / regex / data structures (stdlib, hashbrown, btrees, bigint, petgraph, yada) / nom (parser combinators) / rustls, hmac, aes, crc32, md5, sha256, rsa (crypto) / wasm (webassembly interpreter / JIT) / rand_chacha, fastrand (RNG), UTF-8 validation, nalgebra/ndarray/faer-rs/rust-num, polars (iffy), some kind of interpreter (wasmi, revm-interpreter, starlark, boa (js)), some kind of JIT (cranelift), httparse
- Goal is a starting point for baremetal development on Chipyard SoCs (with APIs for 'syscall' emulation).
- Make baremetal programming much more productive and fun! Demonstrate benefit by porting existing benchmarks and making it so easy to create new ones.

Very limited set of benchmarks that we can run baremetal right now and they aren't very intensive anyways.

Extraction of benchmarks from sampling crates that use base crates and seeing their call site usage and argument distribution.

Microbenchmarks for extracting uArch parameters (memory bandwidth vs latency plot for loaded vs unloaded system, core-to-core cache line bouncing/access latency, memory bandwidth stress tests, ROB size, LSU size, nop elision capability, max retirement/cycle, figuring out number and type of FUs, branch predictor history benchmarks, )

Finally usage of shrinkwrapped Rust binaries running on Linux as full fledged benchmarks.

https://benchmarksgame-team.pages.debian.net/benchmarksgame/measurements/rust.html

## Understanding Things

- riscv-isa-sim
- riscv-tests
  - ISA tests
  - riscv-env
  - rv64ui-p-simple
  - understanding HTIF
  - running on spike
- BareMetal-IDE
  - 

Existing things
  - riscv-env
  - syscall.c in benchmarks
  - libgloss-htif
