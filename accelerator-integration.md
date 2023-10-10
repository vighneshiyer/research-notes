# Accelerator Integration

## AuRORA

### Draft Talk Feedback

- Superficial stuff
    - Using too many color schemes, try to only use a few colors
    - Arrows and highlighting in slides all use different colors, keep them consistent
    - Add page numbers on every slide
    - Add the artifact badges on the first slide
- High-level stuff
    - The intro is too repetitive - simplify it
    - There is too much text on the slides, use more figures and sparsify the existing text
    - Be more excited while giving the talk lol
    - The numbers on the slides are too small - people can already read the paper - highlight specific things and only include a few numbers in the slides
- Specific stuff
    - The key result isn't very clear - what is the aha moment?
        - Present the challenges quickly. Then explain how the requirements of the aurora system address the challenges.
        - How does the full stack thing you present actually address the problems you discussed? How are the requirements layered in the implementation?
        - It isn't clear why the AuRORA technique is better than CPU-coupled accelerators or MMIO accelerators - what problem does it really solve?
    - On the timeline figure for how AuRORA distributes work across accelerators / workloads: it is not clear what this is showing
        - Compare the timeline for AuRORA to prior work (show the thread migration overhead and the overhead of the framework for other work and distinguish it from AuRORA). What makes AuRORA special?
        - Just looking at the basic flow of how work is distributed and accelerators are acquired and released isn't sufficient
            - What makes AuRORA better than other things? (e.g. prior work isn't sufficiently adaptive, has too much overhead, doesn't meet the deadline)
    - Why is AuRORA better than MOCA and Vitair? It isn't clear where the benefits are coming from.

### Questions

- What are the differences between AuRORA and MMIO mapped accelerators again? They seem very similar except for using ISA extensions vs MMIO access to program the accelerator. A similar acquire/grant flow is possible with MMIO accelerators too. There doesn't seem to be any fundamental benefit to the ISA-level programming model vs MMIO. You can just as well have an accelerator virtualization layer in RTL as a MMIO device (and it is probably more flexible than the RoCC ISA too).
- How is thread migration supported? Are accelerators aware of which threads have access to them? How is thread preemption done?

### Comparing CPU-Coupled, MMIO, and ReRoCC Approaches

- The benefits of the CPU-coupled approach are programmability and ability to use the CPU's TLB/PTW
    - The cons are limited opcode space and the core-per-accelerator architecture
- The benefits of the MMIO approach are decoupling accelerators and cores
    - The cons are that you need an IOMMU and synchronization of its TLBs with the core's
- Make it clear that ReROCC combines the benefits of both while mitigating the cons of both!
