## ghOSt: Fast and Flexible User-Space Delegation of Linux Scheduling

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

"Kernel schedulers are hard to implement, test, and deploy" vs userspace schedulers. Schedulers can make a big difference in system utilization, tail latency, and throughput, especially when they can leverage application-specific semantics such as event handling, cooperative yielding, and inter-thread communication patterns.
Using a custom OS image to implement application-specific schedulers is often too much deployment and devops overhead to cope with, but custom schedulers have been proven to improve application performance.

#### Summarize the paper's contributions. length: one paragraph.

The authors present ghOSt, which is a new Linux kernel API to offload scheduling policies into a userspace process - the scheduling mechanics remain in the kernel, while the scheduling policy is in userspace.
The ghOSt userspace process receives messages about thread creation, blocks, and wakeups from the kernel, and it sends scheduling decisions to the kernel.
They demonstrate that ghOSt has a marginal performance overhead over a kernel scheduler, but is far more flexible and easier to make changes to vs kernel scheduler hacking.

#### Summarize the weaknesses of the paper. length: one paragraph.

The authors claim that userspace threading (green threads) isn't sufficient to build a custom and flexible scheduler for a specific workload because it can't ensure that OS threads are pinned to cores unless you give up core sharing among multiple applications.
However, this claim is weak since most dedi servers run only one application (with some background daemons too). Furthermore, userspace threading is often aware of application level semantics (unlike OS thread schedulers) that lead to better scheduling policies.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

Putting logic in the kernel is annoying because you can't unit test any of it.
Testability of schedulers is a very desirable property that the authors emphasize, but don't delve too deep into.
It would be cool to develop a tool that can fuzz a scheduler by injecting synthetic thread spawn/wakeup/halt data into it and seeing how it responds with scheduling decisions.
Also interesting is to correlate the results of fuzzing and directed testing of the scheduler in isolation to the actual performance of the scheduler on a real application.
Can we extract thread spawn/kill behavior from an application, unit test those traces on many scheduling algorithms, and automatically tune those policies for the application without ever having to run the application and scheduler together?

## A Case Against (Most) Context Switches

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Context switches are expensive and lead to (potentially) unnecessary overheads related to cache/TLB warmup/invalidation and state saving + restoration.
The existing SMT support in cores is limited to multiplexing 2-8 software threads on a single core's resources - the OS sees each SMT thread as just another hardware core.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose adding many (100s to 1000s) hardware threads to each physical core where software is in control of which hardware threads can run.
They claim that this can eliminate the need for IO interrupts, eliminate OS syscall overhead, make blocking IO easy, and provide isolation for hypervisors and guests.
They propose some ISA extensions that an OS and user threads could use to implement their hardware thread management scheme.

#### Summarize the weaknesses of the paper. length: one paragraph.

They claim that large multiported register files such as the ones on GPUs mean that it is reasonable to have a large multiported register file for a CPU core that supports 1000s of hardware threads - I think this is insane; the PRF is already a big chunk of area in a modern CPU and this proposal would blow that area up even more.
They claim that higher level caches can be used to store state for 1000s of hardware threads, but don't they realize that this would crush the cache capacity?
Managing fairness in thread scheduling with hardware controlling thread execution is also difficult.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

It is an interesting proposal and would benefit from a prototype implementation both in a functional model (to understand how the thread programming and kernel would look like in practice) and in RTL (to evaluate whether the author's proposal is physically realizable).
A small case study showing the implementation caveats of this proposal would be useful.
