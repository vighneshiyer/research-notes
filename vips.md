# VIP Development

## 1/31/2023

- Past week, reviewed the TL spec and the TL-UL protocol
    - Very basic, 2 channels, A channel goes from master to slave, D channel goes from slave to master
    - Memory read = Get (A) -> AccessAckData (D)
    - Memory write = PutFullData(A) | PutPartialData(A) -> AccessAck(D)
- Earlier implementation of a TileLink VIP and fuzzer: https://github.com/TsaiAnson/verif

### Vighnesh's TODOs

- Republish recent versions of rocket-chip, rocket-dsptools, and dsptools itself
    - Do it under my domain
- Construct a basic TLRAM example for TL VIP work
- Draft a project proposal for you that's based on building a hardware datatype library for Scala

## 1/25/2023

- tapeout, physics 7b + 137a, ee 117, 105
- SURF, ...

- Read through the TileLink spec (only from the start to Ch. 6 - TL-UL) (https://static.dev.sifive.com/docs/tilelink/tilelink-spec-1.7-draft.pdf)
    - Memory interaction primitives: Read, Write, AtomicCAS (compare and swap), ...

    ```scala
    sealed trait MemPrim {
        def toTLPrim(): TLPrim = {
            ???
        }
    }
    case class Read(addr: BigInt) extends MemPrim
    case class ReadResponse(data: Seq[Byte]) extends MemPrim
    case class Write(addr: BigInt, data: Seq[Byte]) extends MemPrim

    sealed abstract class TLPrim(dataWidth: Int)
    case class Get(data: BigInt, addr: BigInt, corrupt, ...) extends TLPrim
    // ... all the various tilelink message types
    ```

    - Develop a data type that models these memory interaction primitives
    - Write a function that can convert between a Read/Write and the TileLink transaction primitives (TileLink messages)
    - Extra: Write function to convert between same memory primitives and the AXI4-Lite transaction primitives
- Develop a VIP that can drive TileLink transactions into the DUT (some TileLink module - maybe a RAM, width converters, buffers, ...)
- TODO: produce a repo that elaborates a TileLink module out-of-context so it can be tested using chiseltest
