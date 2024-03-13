# ML Compilers

## Microkernel vs Direct Emission / Optimization Investigation

Hypothesis: The performance of both library-level kernels (e.g. BLAS operations) and operator graphs (e.g. ONNX models of ML networks) will be higher using compilers that optimize and directly emit code for the target architecture (e.g. x86, AVX, SVE, RVV) vs compilers that implement operators in terms of hand-written microkernels.

Motivation: Jerry's debate with Qualcomm people on the RISC-V mailing list about the AME ISA extension. One party argued that since existing compilers (seem) to use vendor-provided microkernels during code generation, there was no need to worry about encoding new 1D ops in AME, since all the microkernels would be rewritten anyways. Jerry argued that it's better to reuse RVV for 1D ops within the AME since compilers will generate code directly for that target. Who is right?

Preliminary investigation: Investigate how each of LA/ML compilers work internally: are hand-written microkernels used (BLIS), are they synthesized (Exo/Halide), or is optimized code emitted directly without lowering to microkernels (IREE? maybe).

Tools to evaluate:

- Autovectorized naive code (via LLVM autovectorizer)
- BLAS implementations (MKL, OpenBLAS, BLIS)
- cuDNN, cuBLAS, TensorRT
- Intel DNNL
- OpenXLA/IREE (from JAX jit/aot)
- TACO
- Halide/Exo
- TVM
- ONNX Runtime (with/without MLIR optimizer backend)
- Pytorch Glow
- TensorRT (not really suitable for evaluation since it is quite opaque)

Workloads and Benchmarks: TBD - start with specific kernels (e.g. GEMM, CONV) and move to full networks.

Platforms: x86 (+ AVX), ARM (+ SVE/SME), RISC-V (+ RVV) (I think we should exclude GPUs for now)

More generally: No one has really done a comprehensive survey and benchmark of all these random tools floating out there. It isn't clear why some perform better than others and the limitations of each framework are also hidden. Furthermore, it is not at all clear how these compilers work internally - what is the actual complexity in integrating a new piece of hardware or a new ISA extension? What optimizations can be generally applied, and which ones require hand-holding? 

We want such an investigation to motivate the correct ISA-level design of RVV and AME, and eventually build an implementation and tape it out.

Prior work:

- [The Deep Learning Compiler: A Comprehensive Survey (Aug 2020, out of date)](https://arxiv.org/pdf/2002.03794.pdf)
- [Enumeration of various tensor compilers](https://github.com/merrymercy/awesome-tensor-compilers)
- [RVV support in IREE](https://medium.com/@rednoahhsu/ml-compiler-for-risc-v-vector-1960abd1626b)
  - [IREE RVV Docs](https://iree.dev/building-from-source/riscv/#optional-configuration)

Questions from Joonho:

- I'm not sure how this will result in ISA level insights though.
- For matrix operations, wouldn’t we be exposing a set of primitive operations and regardless of the compiler framework, wouldn’t it have to target these primitives?

> Another question is whether existing compilers can efficiently discover optimal blocking/unrolling strategies for mapping high level operators to low-level ISA instructions? I feel we don't really know the answer to this. And yes it also motivates new instructions based on our findings - if things are too control heavy we can move those into hardware

- I guess the question becomes: what are the correct primitives to embed as instructions.
  - Should we be using macro instructions (the hardware has fixed logic to support macro operations, kind of like the Amazon thing)
  - Should the instruction semantics be more fine-grained? If we expose fine-grained semantics, how would that affect the design of these frameworks? Would they be able to perform more aggressive optimizations via better operation interleaving?
  - Would this benefit (more freedom for compilers + simple HW implementation) surpass the benefit of macro-instructions (less instructions executed + simple compiler frameworks + compilers can be agnostic of the hardware which eliminates the need for certain optimizations such as software pipelining)?

- My gut feeling is that for linear algebra operations, the compiler should be able to generate efficient kernels by targeting fine-grained primitive operations (scalar, vector, 2D ops) as the number of operations are fairly limited, and techniques such as tiling and software pipelining are well understood.

> We need to validate that gut feeling - I'm not sure whether those kernels can be generated or if, in the end, some manual tuning is required (or outright explicit microkernels)

- Wouldn’t the higher level execution graph level optimization have a higher impact on performance than the lower level micro-kernel performance? So again, will this lead to ISA/hardware level insights? Am I missing something here? Still, I think it will lead to compiler level insights and guide us on what the current compiler frameworks are missing and how we can do better.

> High level graph optimization often will involve multicore and multi-accelerator stuff - likely there are hardware bottlenecks to resolve here. But furthermore, we don't even understand the breakdown of what optimizations contribute what to overall performance uplift - a thorough understanding will be valuable on its own
