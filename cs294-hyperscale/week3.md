## Azure Accelerated Networking, NSDI 18

https://www.usenix.org/system/files/conference/nsdi18/nsdi18-firestone.pdf

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Networking stacks are doing more and more work (VPCs, routing, ACLs, packet filtering, QoS) and are wasting hypervisor CPU cycles that could be spent on application logic.
Also, customer VMs would like to use SR-IOV to bypass the hypervisor overhead when communicating with the NIC, necessitating moving SDN functionalities to the NIC / hardware.
The SDN functionalities are complex and stateful - to maintain maximum flexibility, new flows should be initiated with the host hypervisor's SDN algorithms and then cached on dedicated hardware.
The desire to use SR-IOV means that either all SDN functions must be implemented in the NIC otherwise any non-implemented function must be done in the hypervisor and that forgoes all the performance benefits - the ASIC SDN approach won't work.

#### Summarize the paper's contributions. length: one paragraph.

The paper considers all the alternative implementations of SDN (ASIC NICs, Multicore SoC NICs, FPGAs, host cores) and makes the case for FPGA accelerated SDN which provides a balance of host-core offload, high throughput, a path to scaling, and sufficient flexibility.
They present the Azure SmartNIC implemented with a bump-in-the-wire FPGA between each host slice's NIC and the TOR switch.
The control plane for the SDN functionalities is retained in the hypervisor, but the data plane is moved to the FPGA - to the VM, the FPGA appears like a SR-IOV NIC.
The authors demonstrate that VMs can saturate the NIC bandwidth with zero CPU utilization on the hypervisor side and only a single core using SR-IOV, and that the SmartNIC hardware has lower latencies and variability than the host-core-based SDN.A
The SmartNIC is capable of being serviced while retaining the state of all TCP connections in the VM.

#### Summarize the weaknesses of the paper. length: one paragraph.

The authors claim that their SmartNIC design can scale easily to 100-400 Gbps networks as datacenter networks evolve, but there is no clear argument for this - what is the limit to packet-level parallelism on FPGAs?
Are there debuggability issues when the data plane is switched entirely to FPGAs? Do we still have the same level of trace data we can use for debugging network issues? The authors make a point that the most common iteration on the SmartNIC RTL involves adding new counters and debug/trace instrumentation.
What are the remaining bottlenecks in the system? Are we close to the theoretical minimum VM-VM latency? What is the cause of the still occasional latency spikes (even though they are much lower than the previous SDN's latency spikes)?
The authors didn't mention which types of SDN functionalities are not supported or would be difficult to support using their SmartNIC architecture - do such functionalities exist and is their use growing over time?

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

Given the SmartNIC RTL, one could do a more detailed host and FPGA time-synchronized profiling to answer some questions: what are the remaining bottlenecks in this system, what is the primary cause of the very tail latency spikes, what are the worst VM/hypervisor/FPGA/network interactions that could happen (that lead to degraded performance).
If we can model an even higher-throughput NIC (100-400Gbps), then we can also answer the question as to whether the SmartNIC architecture will scale well (and also stay within the PCIe card power envelope, which may be the hard part).

## The nanoPU: A Nanosecond Network Stack for Datacenters

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Webapps commonly dispatch many RPCs for a single request and the tail latency / throughput of an app is bottlenecked by the speed of sending/receiving and processing these RPCs.
While some RPCs resolve in milliseconds, others have data in microseconds, so it is critical to optimize how they are handled (since their latency is on the same order as a context switch or DRAM/PCIe access).
The bottlenecks in serving RPC responses are NIC-CPU (PCIe) latency, context switches / interrupt serving, kernel/userspace copying, and traversing the memory hierarchy to get the response packet to the L1 cache/RF.

#### Summarize the paper's contributions. length: one paragraph.

The authors propose and implement a SoC architecture combining NIC, custom/modified RISC-V host cores, and a regular memory hierarchy that achieves very low RPC service times by landing nanorequest responses directly in the core's packet-word buffer/register file, bypassing the rest of the SoC fabric.
They evaluate their RTL implementation using Firesim on several datacenter applications and demonstrate a context switch latency of 50ns (with a baremetal "OS").
They also show various other results about tail latency, roundtrip latency, throughput, the usual stuff.

#### Summarize the weaknesses of the paper. length: one paragraph.

Are nanorequests (RPCs with sub 1uS latency) actually present in large quantities in common datacenter applications such that specialized hardware to handle them makes sense? Even if they are present, is the *application-level* bottleneck serving the responses for these RPCs?
The authors assume that colocating the NIC on the same die as the core is a reasonable design point, but this doesn't seem to be a direction that CPU vendors are moving in - why?
Is there any commercially-relevant flexibility that is lost when integrating the NIC with the SoC (think about upgrade cycles, servicibility, and vendor dependence)?
How does the nanoPU architecture interact with TCP offload hardware - does the new bottleneck become the userspace TCP stack if the host core is responsible for writing responses to a nanorequest?
Can prefetching packet buffers provide enough latency hiding to make the traditional architecture competitive with nanoPU (especially when bottlenecked by RPC message deserialization)?
What flexibility is lost when doing hardware thread scheduling vs kernel thread scheduling? What are the limitations of the HTS (e.g. can it handle thread-local variables properly)?
What is the complexity in integrating the HTS with Linux? The authors only evaluated their own 'kernel' which they completely control.
Precise exceptions become more difficult to handle when reads/writes to a GPR can cause side effects - the authors easily address this with a short queue for an in-order core, but what complexity will come about with a more complex core?
Why can't the netRX/netTX GPRs be instead implemented as CSRs allowing gcc to use all the GPRs as usual? Is the performance overhead of getting the 8B packet-word data in the memory stage vs RF stage that significant?

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

Attempt to prototype the nanoPU architecture using a more complex out-of-order core and sketch out what changes are necessary to support pipeline flushes / precise exceptions.
Integrate the HTS into the Linux scheduler and understand what additional offloads are necessary, and if this idea is even viable.
Attempt to max out the performance of the existing solution (PCIe attached NIC to SoC memory bus) - can intelligent prefetching and better scheduling give most of the benefits or are we just bounded by PCIe latency?
