# SLICE Winter Retreat 2024

## Feedback During Poster Session

### TidalSim

- Motivate why we need this smarts and simpoints hybrid sampling
- Talk about IO devices and why this matters for industry given performance modeling teams
- Perf models are often done in industry for correlation to make sure the RTL matches, written by two different teams
- Brucek suggests using GPU large batch perf simulation for using many many parallel simulations (one issue is GPU accelerated simulation relies on all simulation state being on the GPU - having a host-hosted testharness is a limiter)
- Perf models need to just show trends but they also need to capture the gradient correctly, perf models are still valuable for fast iteration
- We need better error heuristics for Tidalsim, how can we estimate the error for a given interval? We need to know when we have little faith in a given interval, and maybe opportunistically run it in detailed simulation to confirm our error bias.
- We need to do dynamic simulation where the runtime behavior depends on timing behavior, we need to do actual tidalsim (movement between functional and detailed simulation)
- Find a way to integrate power management (DVFS / power gating) so it works and can be modeled in sampled simulation
- Eventually you will be bottlenecked by functional simulation performance! All the instrumentation and embeddings you have to add, will eventually handicap you! You can't easily resort to JIT-based functional simulators to mitigate this - and it only gets worse for multicore / heterogeneous SoC simulators...
  - **Solution**: Just like in RAMP Gold, use FPGAs for functional simulation and warmup models and embedding and checkpointing, BUT, do detailed simulation on the host machine. Interesting... the opposite of the typical scenario (functional sim on host, detailed RTL sim on FPGA)! Leverage high PCIe bandwidth from the FPGA to saturate RTL simulations on the host.
  - On the FPGA we can instantiate many functional cores and many different cache and BP functional models that are warmed up all at the same time

### Krste's discussion on research topics in computer architecture

- Big-little core architecture QoS optimization (how to make sure cores don't starve each other, scheduling stuff, idk about specifics here)
- Power management (better frontend abstractions, better verification, better evaluation/simulation techniques)
  - Build a taxonomy of prior work in power management primitives, lots of verif stuff to do
- Dealing with soft and hard errors - no clear direction for academics to push on however
- Simulation is critical - we need to continue to push on this
- Keep pushing with accelerators and SoC integration methodology (beyond MMIO) and multicore systems
- Thread synchronization primitives, we still communicate via memory, there is better option to do core to core?
  - Cray machine had core shared regfile to share task details among cores via registers, not memory
  - We have so much bandwidth on chip now but core to core bandwidth is still like 512 bit bus! Why is there this limitation? Can we lift it?
- On selling processor IP as SiFive
  - Workloads and benchmarks are important, but hard to convince people, right now people still ask for Dhrystone and Coremark scores and other useless stuff like geekbench
  - Trying to leverage RISC-V Android and browser benchmarks is critical but very difficult

### Leveraging a Full Stack Profiler for Autotuning

- Using profiling information from sampled (or full system-level) simulation to drive black box autotuners
- The question is about what feedback mechanisms autotuners can work with. If they had access to detailed uArch metrics/traces of the kernel they produce, could they leverage that information when searching the kernel optimization space (to do this more efficiently)?

### Generating Simulation Tools from ISA Specs

- In general, work on formalizing specifications is interesting. Can we generate the English spec (e.g. an ISA spec, or a bus protocol spec, or UCIe, etc.) purely from the formal model of the ISA? How do we deal with notes/explanations/organization?
  - Consider formalization not only for verification, but also for ease of implementation
- Can we use the formal spec of an ISA to generate a QEMU-like (JIT'ed) emulator and a spike-like high-performance interpreter? Can we also generate checkpointing and co-simulation logic? What additional formalization is required for things like IO devices, PLIC, etc.?
  - Some people have tried this before (see [The Vienna Architecture Description Language](https://arxiv.org/abs/2402.09087))
  - We should try to take the sail riscv model and generate interpreters / dynamic binary translation based simulators (using LLVM or C emission or whatever to make it easy and not have to encode x86 (host ISA) semantics too).
  - There is of course some unspecified behavior in the ISA spec. How do we specify that behavior when we want a concrete simulator? This is one advantage that the Imperas people have (over spike) - they have concrete simulators for specific RISC-V IP with their own quirks and implementation of undefined behavior.
- In general, **we should generate simulation models**. This would allow us to safely break away from spike eventually if we want to heavily customize the outer step loop but make sure the inner step loop is spec compliant

### Presentation Methodology

- Write papers and qual talk and presentations **as a designer working on a problem and solving each piece of it** (story-driven writing)
  - Example: I made rtl change, how do I see impact? I can do this... But it has this problem. Let's try this (embedding) based on this observation (phase behavior). Then let's try rtl injection. Then we see these results. Why are the results bad? Let's analyze mkpi between full rtl and TidalSim. Oh we see that the caches aren't being warmed! How does warming change with different warming intervals? Can we also simulate intervals around those that are far from centroid? Then how can we do warming? What data structure do we need? How much the is the error reduced by? Basically don't talk about implementation dumb stuff, just talk about a story about developing a tool and tell the story through experiments and plots

## Industry Feedback

- [SLICE Wiki link with detailed notes](https://slice.eecs.berkeley.edu/wiki/retreats/2024winter/feedback)

### Qualcomm

- Generalize the constrained compute use case for robotics to other domains like RF base stations, reduce robot form factor even further like with bees and introduce more constraints, push very far
- Do we really need new hw accel for use cases or is existing stuff enough, what is the bottleneck today?
- Chiplets, tapeout is good, what do you learn?
- On device ML is still not really here, efficient ml inference. More fp8 vs int8 inference comparison would be good. HW accel for compressor has quite high overhead which may not be seen in sw impl. 

### NVIDIA

- Like generators and flows, but evidence is not enough to get adoption for new design languages, how does this solve the verif problem? What about interop?
  - Let's see more generators and flows for arch prototyping and research, not just for tapeout. Need more insights from generator design spaces. Physical design feasibility and power sim with glitches is important, need to show we can do this.
- GPU accelerated simulation, tidalsim, using LLMs for EDA.
- Need more progress getting closer to industry flow, likes AMS and analog, more AI in these flows.
- Quantization is very cool, thanks Coleman, SW opt and next gen HW. Sparsity, can we get different architectures and analyze actual sparsity and look at PPA carefully, actually do this!
- For robotics, look at end to end use of power for small robots, low voltage tricks and circuits stuff, multi power domain stuff, need full stack analysis

### Intel

- Need more industry collab and joint work, extract IPs from Berkeley from Chisel and Chipyard into industry.
- Chiplets and ucie stuff, Chipletyard is one part of the solution, need to blend commercial EDA tools too.
- Like behavioral and RTL models mixed together, tidalsim and sonicsim, be careful when creating arch models back ported from rtl model.
- Love to see intel16 tapeout, ReRoCC is good for scalability and in tapeout, sram22 very exciting for macro compilers, unique sram you can't get from foundry.
- Functional bugs, tool constraints, physical issues, look at these and evolve your curriculum.
- LLMs for codegen is good. Like LTL assertions into MLIR, good. Not much codegen for multi accelerator integration, LLMs codegen for PPA with rtl sim is important too.
- Zoomie is good, more users would be good, how is the experience?
- Firesim on Intel fpga is good, good that you're moving away from cloud FPGA lock in.
- Intel 16 sram is good, can we propagate it to other users?
- Like squeeze LLMs, compare against qlora methods.
- Like vortex as external IP and forcing gemmini into it. Like stellar, want to see more work beyond Hasan's graduation. More optimization passes and dealing with Stellar's verification problem.
- Likes metalift usage of LLMs. Generalizing RoSE for sim and real and illixir, likes to see progression for fast iteration. Forces issue of HW co-design.

### Google

- Already entered into wiki
- Want chiplet security and management stuff, want help on open source advertising?
- Need to brainstorm new industry standards.
- Lots of simulation work and tools, this is good, spatial and temporal cosim, need async multi threaded workload support. Connecting pre and post silicon evaluation tools, long term.
- Difficult to replace SystemVerilog since the new tool has to everything the old tool does and some things better. Hard to ask for adoption of tools like Chisel. Need a compatible complement to Verilog. Something that can ingest Verilog or maybe create an IR representation of a Verilog circui and black box for it.
- Like LLMs for tooling. Interpretability and generalization of LLMs is still important and we want to see specialization for system HW problems.
- Google wants to give a prepared industry presentation before we do our smaller discussions, good idea. Summary email at the end of each day would be good

### Apple

- Like tapeout and bringup class, like undergrad participation, likes to see the generational TAs, wants to know what chips and lessons learned from each class, just a slide presented to industry and future students.
- Academics should be forward thinking, you can also publish failures. Keep working on experimental stuff, wants to see more of that.
- Wants more work on tool stuff, early arch evaluation, industry can see how to do similar things. PPA tradeoff analysis
- Likes power modeling, can we apply this to all research presentations? Need to show these in all slides, PPA tradeoff discussion.
- Likes LLMs, can it be part of design flow methodology? Leverage academic ability to fail! Chiplet design methodology is important. Security is important too, wants to see more.
- Verification, research emphasis is low, focus on methodology, test generation tools, fast functional convergence, need to get to stable functionality very fast in design cycle, corner case exploration, drilling down into corner cases in search space.
- On co-design, more presentations please. Not just discussion. Want PPA tradeoff analysis with software design.
- Talk about the biggest challenge in your project!

### AMD

- Likes chiplets and UCIe interface generators, likes attempt to reuse existing chiplets, but might be tough. Trying to meet industry spec is hard but not essential. Just have a standardized platform that other universities can use. Make Berkeley the leader in this area and lead others.
- Likes 3d stacking research, wants someone to continue research after Harrison. Likes the methodology and forward thinking push wrt the 3D generator stuff.
- Like TidalSim, some limitations to the RTL sim approach for uarch development, there is value for SoC arch exploration from stable IPs, accurate PPA estimation is critical, we don't have capability in that aspect. More complex benchmarks please.
- Likes hyperscale chip with 2 chips, can we leverage chiplets there for larger chips?
- Likes student tapeouts and ugrad participation and demos, likes these special skills sent to undergrads.
- Likes Saturn, Gemmini, Maverick, others, more learnings would be good.
- More power estimates with PPA analysis is very good, wants more of this, we want quantification, perf/W is more important that pure performance. We want to see more work in pre-silicon PPA estimation early on in the design cycle.
- DNN accelerator and core/cache synchronization is critical, need to evaluate area tradeoffs

### Cornell

- Likes ugrad involvement in research, likes tapeouts, what is the research value of this tapeout? What are you learning? Likes bench testing of tapeouts, using silicon on robots is cool.
- SLICE can push dynamic and formal verif, we have unique opportunities. Berkeley challenges conventional wisdom with Chisel and Chipyard, build a framework and use it yourselves.
- Don't settle for flow tools, push Hammer and dogfood it. We made emulation work with Firesim, we used our own stuff! Need more of this on dynamic and formal verif, need to use in teaching tapeouts
  - We have this opportunity, don't settle for just bare metal tests on SoC-level and challenge conventional wisdom
- SLICE lab can make an impact on foundational models for SW/HW co-design. How can we transform the way we do this full stack?
