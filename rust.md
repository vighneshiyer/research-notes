# Rust for Baremetal RISC-V on Chipyard SoCs

## Motivation and Goals

### Background

The [existing baremetal programming environment for RISC-V Chipyard SoCs]() is quite bare-bones and isn't ergonomic to use (C is painful).
There is no reason we can't have an allocator, clean debug APIs, and pthread-like multithreading in a baremetal environment.
It would be nice to avoid Makefiles or random CMake scripts and use a clean `cargo` build setup instead.

Currently, there is a very limited set of benchmarks that we can run baremetal and they don't contain things you would expect to see in real software (complex data structures, memory bound algorithms, bytecode interpreters).

Existing baremetal benchmarks
  - Coremark, riscv-tests benchmarks, rvv-bench, embench, mibench
  - Nothing that great imo

### Goals

- Rust-based baremetal programming environment for RISC-V (specifically our Chipyard SoCs w/ htif host tethering)
- Goal is to get many [no_std] rust libraries working cleanly with RISC-V and leverage them to generate a bunch of useful benchmarks we couldn't do otherwise (would require random C++ libraries running on top of an OS).
  - crossbeam no_std/alloc primitives for multithreaded baremetal programming
  - (de)compression / serde (json, bincode, flatbuffers - iffy) / regex / data structures (stdlib, hashbrown, btrees, bigint, petgraph, yada) / nom (parser combinators) / rustls, hmac, aes, crc32, md5, sha256, rsa (crypto) / wasm (webassembly interpreter / JIT) / rand_chacha, fastrand (RNG), UTF-8 validation, nalgebra/ndarray/faer-rs/rust-num, polars (iffy), some kind of interpreter (wasmi, revm-interpreter, starlark, boa (js)), some kind of JIT (cranelift), httparse
- Goal is a starting point for baremetal development on Chipyard SoCs (with APIs for 'syscall' emulation).
- Make baremetal programming much more productive and fun! Demonstrate benefit by porting existing benchmarks and making it so easy to create new ones.

Extraction of benchmarks from sampling crates that use base crates and seeing their call site usage and argument distribution.

- Microbenchmarks for extracting uArch parameters (memory bandwidth vs latency plot for loaded vs unloaded system, core-to-core cache line bouncing/access latency, memory bandwidth stress tests, ROB size, LSU size, nop elision capability, max retirement/cycle, figuring out number and type of FUs, branch predictor history benchmarks, )
  - See the characterization done by chipsandcheese people
  - Also see various uArch diagrams on twitter that have been reverse engineered
    - https://gist.githubusercontent.com/travisdowns/00e87165356a0e698b49d6cdbf091dd5/raw/5d73ec198acec71cef60b91d67456e476168f742/M1%2520Explainer%2520070.pdf
  - [M1 explainer](https://gist.githubusercontent.com/travisdowns/00e87165356a0e698b49d6cdbf091dd5/raw/5d73ec198acec71cef60b91d67456e476168f742/M1%2520Explainer%2520070.pdf)

Finally usage of shrinkwrapped Rust binaries running on Linux as full fledged benchmarks.

https://benchmarksgame-team.pages.debian.net/benchmarksgame/measurements/rust.html

https://docs.google.com/presentation/d/1H83fB7tNbzhnz4kJ0BNqbgb7xSXqhPYXDmAP-vXa4_c/edit#slide=id.g2896b2792b6_0_84
Yufeng's baremetal tensor library

## High-Level Tasks

- Need to port libgloss-htif to a Rust BSP
- Need to get multithreaded baremetal code working, make it easy to use 'pthreads' like APIs baremetal
- Need ability to use a baremetal allocator
- Produce very clean shrinkwrapped binaries with zero dependencies and zero libc nonsense




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
