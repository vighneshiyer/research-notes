## ghOSt: Fast and Flexible User-Space Delegation of Linux Scheduling

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

"Kernel schedulers are hard to implement, test, and deploy" vs userspace schedulers. Schedulers can make a big difference in system utilization, tail latency, and throughput, especially when they can leverage application-specific semantics such as event handling, cooperative yielding, and inter-thread communication patterns.
Using a custom OS image to implement application-specific schedulers is often too much deployment and devops overhead to cope with, but custom schedulers have been proven to improve application performance.

#### Summarize the paper's contributions. length: one paragraph.

The authors present ghOSt, which is a Linux kernel hack to offload scheduling policies into a userspace process.

#### Summarize the weaknesses of the paper. length: one paragraph.


#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.


## A Case Against (Most) Context Switches

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.


#### Summarize the paper's contributions. length: one paragraph.


#### Summarize the weaknesses of the paper. length: one paragraph.


#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

