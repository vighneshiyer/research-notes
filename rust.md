# Rust for Baremetal RISC-V Chipyard SoCs

## Motivation and Goals

### Background

The existing baremetal programming environments for RISC-V Chipyard SoCs ([riscv-tests benchmarks](https://github.com/riscv-software-src/riscv-tests/tree/master/benchmarks), [gemmini-rocc-tests](https://github.com/ucb-bar/gemmini-rocc-tests), [Chipyard tests](https://github.com/ucb-bar/chipyard/tree/main/tests)) are quite bare-bones and aren't ergonomic to use (C is painful).
[Baremetal-IDE](https://github.com/ucb-bar/Baremetal-IDE) is an improvement, but perhaps we can do even better.
There is no reason we can't have an allocator, clean debug APIs, and pthread-like multithreading in a baremetal environment.
It would be nice to avoid Makefiles or random CMake scripts and use a clean `cargo` build setup instead.

Additionally, we would like a set of realistic benchmarks for microarchitectural iteration that can run baremetal (booting Linux or even using [pk](https://github.com/riscv-software-src/riscv-pk) is limiting).
But currently, there is a very limited set of baremetal benchmarks and they don't contain things you would expect to see in real software (complex data structures, memory bound algorithms, bytecode interpreters).
They include: [coremark](https://github.com/riscv-boom/riscv-coremark), the [riscv-tests benchmarks] (qsort, dhrystone, spmv, towers), [rvv-bench](https://github.com/camel-cdr/rvv-bench), [embench](https://github.com/embench/embench-iot/tree/master), [mibench](https://github.com/embecosm/mibench), and [Baremetal-NN](https://github.com/ucb-bar/Baremetal-NN) ([slides](https://docs.google.com/presentation/d/1H83fB7tNbzhnz4kJ0BNqbgb7xSXqhPYXDmAP-vXa4_c/edit?usp=sharing)).
Benchmarks that are representative of mobile/desktop/server/HPC usecases (SPEC, NPB, PARSEC, Geekbench) require at least an OS and can't be used in the loop of microarchitectural iteration.
Let's try to leverage the `no_std` feature of Rust to make realistic (single-core) benchmark construction easy.

Finally, we want microbenchmarks that can estimate the types, dimensions, and performance of microarchitectural structures (caches, branch predictors, prefetchers) present in an SoC.
A nice and clean set of microbenchmarks would help us verify and hit high coverage on our cores as well as commercial RISC-V silicon.
Many of these benchmarks already exist, but we want to reconstruct, re-characterize, and package them cleanly (the value of this is underappreciated).

### 3 Primary Goals

#### Baremetal Programming Environment

We want a Rust-based baremetal programming environment for RISC-V (specifically our Chipyard SoCs with HTIF host tethering).
This will be a starting point for baremetal development on Chipyard SoCs; a similar concept as Baremetal-IDE, but using Rust.

We want to make baremetal programming much more productive and fun!
And we will demonstrate the benefits by porting existing microbenchmarks and making it easy to create new ones.

Even for benchmarks which must run on top of Linux or pk, we want to demonstrate the ability to create complex, shrinkwrapped, statically-linked, glibc-free Rust binaries.
And perhaps these binaries can run more easily on top of pk than ones produced with the typical C++ gcc-linux cross compilation flow.
<small>Maybe we can even rewrite pk for fun?</small>

#### New Clean-Slate Realistic Benchmarks

Existing baremetal benchmark suites don't even attempt to reproduce similar compute/memory behaviors as real applications.
Instead, they contain low performance, naive code from 20+ years ago that we [can't even understand](https://github.com/embench/embench-iot/blob/master/src/edn/libedn.c), let alone modify.

Our goal is to subsume all existing baremetal benchmarks by using `no_std` Rust libraries and generating useful benchmarks that we couldn't do otherwise (with random C++ libraries running on top of an OS).
Since it is mostly *libraries* and not *applications* that are marked as `no_std`, we need some way to capture the real usage of these libraries.
We can try to extract benchmarks from looking at crates which use `no_std` crates and tracing how the library functions are invoked and with what arguments.

#### Microarchitecture Microbenchmarks

A few microarchitecture blogs (Chips and Cheese, Geekerwan, [@Cardyak](https://x.com/Cardyak), Anandtech) attempt to estimate microarchitectural structures in commercial silicon.
We want to build microbenchmarks that can be used for this purpose.
The algorithms themselves already exist, but it would be cool to see them packaged neatly, and we can characterize our own cores in addition to commercial RISC-V cores.

## Rust RISC-V Baremetal Environment

crt + base linker script
HTIF support
multithreading support
allocator support
many examples
BSP for particular chips, perhaps even riscv devboards

- https://github.com/rust-embedded/riscv

https://openbenchmarking.org/

Other benchmarks like SPEC (cpu, jbb, omp), NPB, PARSEC, GAP, Coremark-PRO, PolyBenchC, MLPerf Tiny (on CPU), MediaBench, Rodinia Benchmark Suite, Geekbench, SPLASH-2, AutoBench, game console emulators, LMbench, Python benchmarks, STREAM benchmark, Speedometer, Cinebench

Existing baremetal benchmarks
  - Coremark, riscv-tests benchmarks, rvv-bench, embench, mibench
  - Nothing that great imo

## Rust-Based Baremetal Benchmarks

### Ideas About Benchmarks

- crossbeam no_std/alloc primitives for multithreaded baremetal programming
- (de)compression / serde (json, bincode, flatbuffers - iffy) / regex / data structures (stdlib, hashbrown, btrees, bigint, petgraph, yada, regex) / nom (parser combinators) / rustls, hmac, aes, crc32, md5, sha256, rsa (crypto) / wasm (webassembly interpreter / JIT) / rand_chacha, fastrand (RNG), UTF-8 validation, nalgebra/ndarray/faer-rs/rust-num, polars (iffy), some kind of interpreter (wasmi, revm-interpreter, starlark, boa (js)), some kind of JIT (cranelift), httparse
- Things we still want: webservers, compilation, solvers (physics, optimization, combinatorial - SAT/SMT), web browsers, graphics, networking / packet processing / protocol stacks), full-featured parsers, document rendering (PDFs), filesystem drivers, kernel space stuff, image decompression / processing / format converters, text editor
- https://github.com/sarsko/CreuSAT?tab=readme-ov-file

- Interesting benchmarks for cloud providers and hyperscalers (I'll use Google as a concrete example)
  - Allocators: TCMalloc
  - Serdes: Protobuf (HyperProtoBench)
  - Stdlib [data structures](https://abseil.io/docs/cpp/guides/container), [hashes](https://abseil.io/docs/cpp/guides/hash), [randomness](https://abseil.io/docs/cpp/guides/random), [string](https://abseil.io/docs/cpp/guides/format) [manipulation](https://abseil.io/docs/cpp/guides/strings), [logging](https://abseil.io/docs/cpp/guides/logging)
  - memcpy
  - DCT (see profiling a WSC paper), compress/decompress, memcpy, RPC, protobuf, allocation, hash, Linux kernel (scheduler, filesystem, networking, ...)
  - SPEC isn't good (small working set size, many hotspots, callgraph structures are shallow, no multithreading, no kernel activity, no networking)
  - Records: RecordIO (Google internal), [Riegeli](https://github.com/google/riegeli), Apache Arrow / Polars-style DF manipulation, reading, and querying
  - Compression: Brotli, Zlib, Snappy, zstd
  - Encryption / crypto hashing: BoringSSL microbenchmarks, RSA, ECDSA-P-256, AES-128-GCM, AES-256-GCM, X25519, SHA-256, SHA-512
  - Non-cryptographic hashing: adler32, absl::hash, CityHash, CRC32C, vhash
  - gRPC library usage + its internal TCP syscalls
  - C++ STL: std::function, std::sort
  - strings: short-string optimizations, std::string, std::string_view (benchmarks are in libc++) (https://github.com/llvm/llvm-project/blob/main/libcxx/test/benchmarks/string.bench.cpp) (https://github.com/abseil/abseil-cpp/blob/master/absl/strings/cord.h) - absl short strings
  - cross-thread synchronization: absl mutex, absl spinlock (hard to model representatively in microbenchmarks)
  - golang: garbage collection, data structures
  - regex: [RE2](https://github.com/google/re2)
  - LA: TensorFlow, Eigen, MKL, libxsmm
  - Video: ffmpeg, x264, VPX
  - Heterogenous boxes: dealing with noisy neighbors on the same node (not in a different VM, but rather multiple processes working on different tasks/responding to different RPCs on the same OS)

- https://doc.rust-lang.org/beta/std/hint/fn.black_box.html
https://benchmarksgame-team.pages.debian.net/benchmarksgame/measurements/rust.html

## uArch Microbenchmarks

- Microbenchmarks for extracting uArch parameters (memory bandwidth vs latency plot for loaded vs unloaded system, core-to-core cache line bouncing/access latency, memory bandwidth stress tests, ROB size, LSU size, nop elision capability, max retirement/cycle, figuring out number and type of FUs, branch predictor history benchmarks, )
  - See the characterization done by chipsandcheese people
  - Also see various uArch diagrams on twitter that have been reverse engineered
    - https://gist.githubusercontent.com/travisdowns/00e87165356a0e698b49d6cdbf091dd5/raw/5d73ec198acec71cef60b91d67456e476168f742/M1%2520Explainer%2520070.pdf
  - [M1 explainer](https://gist.githubusercontent.com/travisdowns/00e87165356a0e698b49d6cdbf091dd5/raw/5d73ec198acec71cef60b91d67456e476168f742/M1%2520Explainer%2520070.pdf)
  - Latency vs CPU core 0 to core N - identification of latencies to jump across clusters with shared L2, to LLC, to global SLC, to ... - depends on the chip memory architecture
  - Memory bandwidth (GB/s/socket) vs latency - identify latency of DRAM fetch into local L1 with varying amount of bandwidth requested by the workload, consider usage of prefetchers (https://www.intel.com/content/www/us/en/developer/articles/tool/intelr-memory-latency-checker.html)
  -


### Benchmark Enumeration

- leverage servo and other rust top-level applications to drive stimulus generation for the baremetal microbenchmarks

#### Baremetal

- Coremark (super old, not representative)
- Coremark-PRO
- mibench / embench (low quality code)
- riscv-tests benchmarks (qsort, dhrystone, spmv, towers) (compute bound workloads)
- rvv-bench (pretty good for RVV)
- Baremetal-NN (pretty good for on-CPU ML workloads)

#### OS Required

##### "Easy" to Run

Easy means

- SPECcpu
- SPECjbb
- NPB
- PARSEC

##### "Hard" to Run

- Geekbench
- Cinebench


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

First steps:
  - riscv-tests single-thread benchmarks port
  - can we generate crt.s from a build.rs? I would like to avoid macros in crt.S and use regular function arguments / config options in the build process instead.
  - rebuild embench benchmarks from their descriptions (don't port their actual code)
  - riscv-tests multi-thread benchmarks port
  - riscv-tests RVV benchmarks port
  - chipyard/tests mt-hello port
  - begin building our own benchmark suite
  - figure out the function call site instrumentation methodology and get the stimulus data distributions
  - build representative benchmark stimulus suite
