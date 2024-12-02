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

For this, we want a simple environment where we can mark an entry point, have a panic handler, have a basic baremetal allocator, we can use HTIF for syscall proxying, and have sensible and minimal bootup code + linker script.
A similar multithreading setup to `riscv-tests/benchmarks` would be nice too: just have separate code paths for each thread pinned on each core, but with the usage of Rust synchronization primitives ([example](https://github.com/rcore-os/rcore-thread)).
If we are ambitious, we can also make true multithreading work in baremetal, which would involve a scheduler, a way to mount/unmount threads, and so forth (still no virtual memory however and only a single process would be supported).

This would also involve setting up [BSPs for particular chips](https://github.com/ucb-bar/Baremetal-IDE/tree/main/platform) and some [RISC-V commercial silicon](https://github.com/rust-embedded/riscv/blob/master/riscv-peripheral/examples/e310x.rs).
Many usage examples would be critical.

The [rust-embedded/riscv](https://github.com/rust-embedded/riscv) project is a very good place to start.
We should use as many things from here as dependencies and contribute upstream.

### Tasks

- [x] Review existing RISC-V C baremetal environments
  - riscv-isa-sim, riscv-tests (ISA tests), riscv-test-env, `rv64ui-p-simple`, understanding HTIF, syscall.c in `riscv-tests/benchmarks`, `libgloss-htif`
- [ ] Use `rust-embedded/riscv` crates as dependencies
  - Get simple hello world over HTIF working
- [ ] Port all/some riscv-tests single-thread benchmarks (dhrystone, median, memcpy, mm, multiply, qsort, rsort, spmv, towers, vvadd)
  - A language model should be able to do 90% of the work of porting this C to Rust
  - When doing this, let's get an idea of what deficiencies exist with Rust's baremetal environment / borrow checker
- [ ] Idiomatically rebuild the above benchmarks using Rust
  - A lot of these can just be expressed by functions/data structures from the Rust stdlib or using a crate
  - Print out some kind of performance metric once the test completes, set a number of warmup cycles and execution cycles and print average runtimes
- [ ] Re-build embench benchmarks from their descriptions
  - Don't just translate C to Rust
  - e.g. for `aha-mont64`, use a crate (e.g. `rust-num`) and just call its implementation (of Montgomery modular multiplication) with the same arguments as the Embench workload
  - For some benchmarks, this might be too painful, in which case just note why and ignore it
- [ ] Port riscv-tests multi-threaded benchmarks (mt-matmul, mt-memcpy, mt-vvadd)
  - Try to leverage the startup code in `rust-embedded/riscv` (I 'think' there should just be a hart id read + loop stripmining approach)
- [ ] Port riscv-tests RVV benchmarks (vec-daxpy, vec-memcpy, vec-sgemm, vec-strcmp)
  - This is a good chance to evaluate the state of Rust's RVV intrinsics
- [ ] Port `chipyard/tests`
  - `mt-hello` should be easy to port
  - Eventually this directory should become Rusty
- [ ] Port some small parts of `gemmini-rocc-tests`
  - Building a `rocc` Rust library would be useful

### Long-Term

- [ ] Begin building our own benchmark suite
  - Reference the Rust libraries and benchmarks below
- [ ] Figure out the function call site instrumentation methodology and get the stimulus data distributions
  - To build a representative benchmark stimulus suite
- [ ] Investigate crt.S generation
  - Can we generate crt.s from a build.rs? I would like to avoid macros in crt.S and use regular function arguments / config options in the build process instead.
- [ ] Random things
  - [ ] Port libgloss-htif
  - [ ] Need ability to use a baremetal allocator
  - [ ] Multithreaded baremetal code

## Rust-Based Baremetal Benchmarks

### Existing Benchmarks by Category

So what already exists in the world of benchmarks?
[Openbenchmarking](https://openbenchmarking.org/) covers lots of typical benchmarks run on commercial silicon, but most of these are hard to use for microarchitectural iteration.

I mark things as **baremetal** or **easy**/**hard**.
**Baremetal** runs baremetal of course (perhaps only an allocator is required).
**Easy** means just a single binary, no GUI, no runtime, mostly shrinkwrapped (no external dependencies), and doesn't depend on any unusual syscalls (probably will run under `pk`).
**Hard** is any workload that isn't easy to get running.

- General purpose integer/FP code
  - [Coremark](https://github.com/riscv-boom/riscv-coremark) (super old, not representative) (**baremetal**)
  - [Coremark-PRO](https://github.com/eembc/coremark-pro) (5 integer and 4 FP workloads, uses multithreading, but can still work baremetal) (**baremetal**)
  - [mibench](https://github.com/embecosm/mibench) (lots of random things) (**baremetal**)
  - [embench](https://github.com/embench/embench-iot/tree/master) (more random things) (**baremetal**)
  - [toddmaustin/bringup-bench](https://github.com/toddmaustin/bringup-bench) (random things, but still relevant) (**baremetal**) ([Todd Austin's announcement](https://www.linkedin.com/posts/prof-todd-austin_opensource-computerarchitecture-benchmarks-activity-7265000332819091456-dI48/?utm_source=share&utm_medium=member_android))
  - [riscv-tests benchmarks](https://github.com/riscv-software-src/riscv-tests) (qsort, dhrystone, spmv, towers) (compute bound workloads) (**baremetal**)
  - [SPEC CPU 2017](https://www.spec.org/cpu2017/) (still considered useful and "somewhat" representative) (**easy** to run)
  - [Geekbench](https://www.geekbench.com/) (widely used and reported, [internals](https://www.geekbench.com/doc/geekbench6-benchmark-internals.pdf) - seems reasonable and easy to replicate) (**hard**)
  - [Phoronix Test Suite](https://www.phoronix-test-suite.com/) (very comprehensive, contains benchmarks in all categories below, SOTA afaic) (**hard** to run most benchmarks in this suite in simulation)
  - [MediaBench](https://cs.slu.edu/~fritts/mediabench/) (very very old)
- Vectors / numerics
  - [rvv-bench](https://github.com/camel-cdr/rvv-bench) (pretty good for RVV) (**baremetal**)
  - [Baremetal-NN](https://github.com/ucb-bar/Baremetal-NN) (pretty good for on-CPU ML workloads) (**baremetal**)
  - [PolyBenchC](https://github.com/MatthiasJReisinger/PolyBenchC-4.2.1) (*likely* **baremetal**)
  - [MLPerf Tiny](https://github.com/mlcommons/tiny) (**baremetal** possible)
  - [BioBench2](https://github.com/reiverjohn/biobench2) (old and irrelevant)
- HPC / parallel numerical computations
  - [NASA Parallel Benchmarks](https://www.nas.nasa.gov/software/npb.html) (uses MPI or OpenMP) (**hard**)
  - [PARSEC (Princeton Parallel Benchmarks)](https://github.com/bamos/parsec-benchmark) (**hard**)
  - [Splash-3](https://github.com/SakalisC/Splash-3) (*may* be **easy**)
- Graphics
  - [Cinebench](https://www.maxon.net/en/cinebench) (CPU 3D rendering, running this in simulation may be too hard) (**hard** to run)
- uArch extraction / reverse engineering microbenchmarks
  - [LMbench](https://lmbench.sourceforge.net/) ([Github](https://github.com/intel/lmbench)) (memory latency/bandwidth ubenchmarks, **easy**)
  - [multichase / multiload](https://github.com/google/multichase) (pointer chasing, latency/BW/loaded latency, **easy**)
  - [mess](https://github.com/bsc-mem/Mess-benchmark) (DRAM BW/latency curves, **easy**)
  - [STREAM](https://www.cs.virginia.edu/stream/) (memory bandwidth)
- Servers
  - [SPECjbb 2015](https://www.spec.org/jbb2015/) (quite representative of backend server workloads) (**hard**)
- Cyber-Physical Systems
  - [EEMBC AutoBench](https://www.eembc.org/autobench/) (mostly irrelevant)
  - [RobotPerf](https://github.com/robotperf/benchmarks/tree/main/benchmarks) (**hard**)

### Ideas About Benchmarks

- leverage servo and other rust top-level applications to drive stimulus generation for the baremetal microbenchmarks
GAP, PolyBenchC, MLPerf Tiny (on CPU), MediaBench, Rodinia Benchmark Suite, SPLASH-2, AutoBench, game console emulators, LMbench, Python benchmarks, STREAM benchmark, Speedometer

- https://gitlab.laas.fr/matana/bench/benchmark-riscv
- https://docs.rs/embedded-graphics/latest/embedded_graphics/
- https://github.com/parsa-epfl/cloudsuite/blob/main/docs/benchmarks/data-serving.md
- https://github.com/google/fleetbench

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
- [DCT](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44271.pdf)
https://benchmarksgame-team.pages.debian.net/benchmarksgame/measurements/rust.html
- https://docs.rs/rkyv/latest/rkyv/index.html (in place data deserialization)
- rug arbitrary precision numbers
- http://www.numberworld.org/y-cruncher/
- Intel's memory latency checker
- LMBench, STREAM, OSU Microbenchmarks, NanoBench, CacheBench (part of PolyBench)

### Rust no_std Libraries

- crossbeam no_std/alloc primitives for multithreaded baremetal programming
- (de)compression / serde (json, bincode, flatbuffers - iffy) / regex / data structures (stdlib, hashbrown, btrees, bigint, petgraph, yada, regex) / nom (parser combinators) / rustls, hmac, aes, crc32, md5, sha256, rsa (crypto) / wasm (webassembly interpreter / JIT) / rand_chacha, fastrand (RNG), UTF-8 validation, nalgebra/ndarray/faer-rs/rust-num, polars (iffy), some kind of interpreter (wasmi, revm-interpreter, starlark, boa (js)), some kind of JIT (cranelift), httparse
- https://doc.rust-lang.org/beta/std/hint/fn.black_box.html

## uArch Microbenchmarks

- Microbenchmarks for extracting uArch parameters (memory bandwidth vs latency plot for loaded vs unloaded system, core-to-core cache line bouncing/access latency, memory bandwidth stress tests, ROB size, LSU size, nop elision capability, max retirement/cycle, figuring out number and type of FUs, branch predictor history benchmarks, )
  - See the characterization done by chipsandcheese people
  - Also see various uArch diagrams on twitter that have been reverse engineered
    - https://gist.githubusercontent.com/travisdowns/00e87165356a0e698b49d6cdbf091dd5/raw/5d73ec198acec71cef60b91d67456e476168f742/M1%2520Explainer%2520070.pdf
  - [M1 explainer](https://gist.githubusercontent.com/travisdowns/00e87165356a0e698b49d6cdbf091dd5/raw/5d73ec198acec71cef60b91d67456e476168f742/M1%2520Explainer%2520070.pdf)
  - Latency vs CPU core 0 to core N - identification of latencies to jump across clusters with shared L2, to LLC, to global SLC, to ... - depends on the chip memory architecture
  - Memory bandwidth (GB/s/socket) vs latency - identify latency of DRAM fetch into local L1 with varying amount of bandwidth requested by the workload, consider usage of prefetchers (https://www.intel.com/content/www/us/en/developer/articles/tool/intelr-memory-latency-checker.html)
