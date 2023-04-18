# Semantic Protocol Compression for NoCs

## Overview

- If we have a schema of transactions, and many traces of real traffic, can we find an optimal encoding/decoding scheme to minimize bus/NoC traffic?
    - e.g. in TL, we have several message types: Get(addr), Put(addr, data), ...
    - Can we design hardware to go from these messages to raw packet bits that can later be decoded?
    - Can we also design a more optimal physical encoding rather than just putting each transaction field on a seperate wire? Can the hardware then operate on this more efficient encoding directly?
    - We will have to trade-off the PPA complexity of the decoder/encoder and the savings we get in bandwidth and network traffic
    - We can't use an online compression scheme since it will take too much area; also we know what type of traffic to expect, so we can use offline profiling and the design of a custom compression network-wide scheme for a given transaction schema
    - e.g. we can use delta compression for addresses and/or we can use entropy encoding for opcodes, and/or magic constants for common values such as all ones or all zeros, or turning one-hot signals into multi-bit signals (e.g. byte-wise mask into an uinteger), we can elide fields that are unused in certain message types. We can have different codecs for master->slave traffic vs slave->master traffic. Encodings may be stateful and reuse addresses/offsets of previous transactions.
    - This type of context-aware (protocol/transaction schema aware) technique should work better than raw packet compression techniques simply because we can take advantage of the user-defined semantics of the protocol
- Prior work: https://ieeexplore.ieee.org/abstract/document/9459512
    - FlitZip: Effective Packet Compression for NoC in MultiProcessor System-on-Chip
    - We want to compare on aggregate PPA cost and benefit and on real RTL evaluation unlike their simulator based eval
    - https://github.com/xingyuli9961/CS262AProject-FlitReduce-NoC_Compression
    - https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=6742954 (NoΔ: Leveraging Delta Compression for End-to-End Memory Access in NoC Based Multicores)

- (Optimal Protocol NoC Packetization): NoC protocol to packet packing (profile based) automatic RTL generation including optimal compression scheme

- Generating Compact Physical Encodings of Transaction-Level Messages
- Can we discover a memory protocol that is more compressible than TileLink / AXI4+ACE / CHI?
    - Discover a common set of primitives that can express the same logical operations as an existing memory protocol, but is more compressible
    - Major problem: how can we prove equivalance and discover rewrite rules to unify 2 sets of primitives
    - Automatic generation of protocol adapters via program synthesis and rewriting
        - Can we automatically synthesize a TL to CXL protocol adapter?
        - Can we formally make sure a protocol never deadlocks and holes in a mapping.

```scala
Get(addr, nBytes)

Read(addr, `nBytes < 8`, beats: Int)

def translate(g: Get): Seq[Read] = {
    ???
}
```

## Log

### Initial Exploration of TileLink TNXS From One Core Rocket (4/15/2023)

```csv
workload,raw_bits_sent,raw_bits_recv,comp_bits_sent,comp_bits_recv,ratio_bits_sent,ratio_bits_recv
dhrystone.riscv.out,29473,75672,12577,15768,2.343404627,4.799086758
hello-world.out,21944,66540,9464,13740,2.318681319,4.84279476
median.riscv.out,25665,80120,11137,16312,2.30448056,4.911721432
mm.riscv.out,243677,388736,102621,78784,2.374533478,4.934199838
mt-matmul.riscv.out,31419,89788,13563,18620,2.316522893,4.822126745
mt-vvadd.riscv.out,400568,810588,170424,163356,2.350420129,4.962095056
multiply.riscv.out,15576,48848,6744,10000,2.309608541,4.8848
pmp.riscv.out,25844,92224,11316,18560,2.283845882,4.968965517
qsort.riscv.out,183139,317008,77603,63632,2.359947425,4.981895901
rsort.riscv.out,1523286,1547248,634262,309680,2.401666819,4.996280031
spmv.riscv.out,298088,1033124,130856,206884,2.277984961,4.99373562
towers.riscv.out,15534,48848,6702,10000,2.317815577,4.8848
vvadd.riscv.out,24409,82484,10649,16756,2.292140107,4.922654571
```

At first glance, it looks like many bits can be saved just from not sending TileLink fields that are zero (e.g. the data field on the A channel when the opcode is GET).
Upon closer inspection, constellation already strips the data field, specifically for this scenario!
So this level of compression is not going to happen.
But, this does reveal that constellation already performs some type of 'compression', but it is ad-hoc - we can build a framework to systemitize semantic compression.

### Conclusions Reached on 4/16/2023

- Raw TileLink interface has unnecessary fields
    - Channels A and D have data fields, but they are only used with certain opcodes
    - Naive TileLink bus widgets probably retain the data fields even if they aren't used in a given TileLink transaction
- If we were to eliminiate this data field when unnecessary, it could yield a compression ratio of 5
- But, constellation appears to not emit the data field (and the mask and corrupt fields too) if they are not necessary or just the default values
    - This means that eliding the data field and mask/corrupt is a kind-of built-in compression already present in constellation
- The first flit of a TileLink transaction consists of its "const" fields and they take up only 50-75% of the bits in the flit
    - This means there's potential to pack more data in this flit
- We don't know how much utilization of the header flit in the packet is present (the flit that's used to route the packet and contains info about origin/dest, channel, VC, QoS, etc.)
    - If this utilization is small, perhaps we could fit the protocol level data into the header flit too
- There is a lot of repetition in these riscv ubenchmarks
    - There are writes that have data set to all zeros
    - The "const" fields in each TileLink transaction are nearly always the same
    - The address fields are off by a small constant from the prior packet (which can be encoded in 2-5 bytes vs 8 bytes)
    - There is opportunity to perhaps pack data into the protocol "const" flit
    - There is opportunity to pack multiple transactions into the same flit
- Tentative conclusion: there is probably some improvements to be made here, but it's hard to tell how much compressibility there really is
    - Q: is the prior work able to already perform decent compression by looking at flits directly rather than the protocol semantics?
        - Q: is there value in semantic compression?
    - Q: can a similar analysis be done with AXI4 + ACE?

### Looking at Prior Work (4/17/2023)

FlitZip and NoDelta seem to be the nominal prior works to examine.
First step is to examine NoDelta at the flit level (only considering flits generated from the TileLink ingress - not considering the head flit added to the packet with routing/VC information).
Someone has a Verilog implementation of FlitZip (https://github.com/Aryan108/Flitzip).

- After implementing NoC-delta in python
    - The results of the paper may not be fake
    - 80 % of the data that was transmitted was actually just zero
    - The compression ratio (**just by the total number of bits sent**) was 1.04 for data that was being sent, while it was around 3.9 for data that was being received. This was because most of the responses had a data field of 0 for 80% of the time among all requests & responses with a valid data field.
    - However, one caveat is that all the papers were using a **128 B** data field which is a bit large (even in modern systems - may have to search for reasonable/popular bus widths). When using a bus width of 64 B, it is impossible to fit the base in the header flit, reducing the amount of compression in terms of flit granularity (which is what we care about).
    - **Another caveat is that all these papers does not consider the granularity of the protocol level-transactions**. That is, in TL, data transfers does not happen in 64B granularity. Each transaction typically encodes 8B of data & utilizes the burst mode to transfer 64B of data. Therefore non-delta compression cannot gain any compression ratio increase in the flit level because, each transaction consists of two flits (header /data) but the base+delta compressed flit will also consist of two flits (header & base / delta). I think we would have to see how if constellation strips out the header field and just sends the data field in burst mode.
    - Probably need to double check the python code tomorrow & reason why there are so much zeros in the data field

### Types of Compression

Try to organize the space of compression algorithms, points of injection (packet-level, flit-level, protocol-level), history requirements into a taxonomy.

```scala
// 1. stateless, single field algorithms
// 2. stateful, single field with history
// 3. stateless, entire flit/packet at once (this is just a specific case of 1)
// 4. stateless, field to field dependency (if opcode is READ, then data is known to be zero / irrelevant)
// 5. stateful, field to field with history
```

Most of these seem redundant - continue to refine.

## Tasks

### Naive Compression Eval

- [ ] Build a Rocket + Constellation (single-node) NoC
    - Simple connection from Rocket to L2 (4 banks) via a small NoC network
- [ ] Validate simple riscv benchmarks can run in RTL sim
- [ ] Add prints for the raw packet traffic going through each NoC node
- [ ] Collect prints from multiple ubenchmarks
- [ ] Parse prints into raw packets sent on each node
- [ ] Run a set of naive compression algorithms directly on the entire packet / the tail flits
    - This is the theoretical peak - it is isn't feasible since the NoC would have to decompress the routing info on each hop to route the packet
    - Only compressing the tail flits is more reasonable since the decompression only happens once the packet reaches its target node (but extra buffering is necessary)

### Protocol Aware Compression

- [ ] Add prints on each TL ingress/egress node to print the entire TL packet data (per channel)
- [ ] Do some manual evaluation of TL transactions on each node (does this look compressible?)
