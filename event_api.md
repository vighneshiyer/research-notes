# Chisel Event API

## Notes from ATHLETE Sync with Si-En (2/8/2024

- post-silicon world, trace buffer for event logs (fixed size, subsample events), don't want to perturb DRAM state because of this - perturb application that's running and perturbing the uarch events themselves
- in-memory format of an event trace (print messages - simulation) (binary format that can work in simulation and silicon)
- event-tracking API can be used for validation coverage too (erase time-axis/resolution)
- tag propgation is often not feasible for post-silicon usage due to area blowup
- only collect events that are valuable for future generation chip
- perf bugs are caught at block or subsystem level, not at SoC top-level - perf targets baked into block-level specs
- event graphs for post-silicon and SoC pre-silicon debug are at system-level

## Random Sketch (Bad Idea)

```scala
trait Event[T <: Data] {
    type Tag = (UInt, Event)
    def getNewTag(): Tag
    def recycleTag(tag: Tag): Unit
    def trigger(tag: Tag, metadata: T, parentEvents: Set[Tag]): Unit
}

abstract class SynthesizableEvent[T <: Data] extends Event[T <: Data] {
    val tag
    def getNewTag(): ???
}

case class FetchMetadata extends Bundle {
    val pc = UInt(64.W)
    val inst = UInt(32.W)
}

class FetchEvent extends Event[FetchMetadata]

case class Commit

case class Retire

case class L1Request
    instId: ???
    addr: UInt
    accessType: Read | Write

case class L1Response
    // no metadata is needed
    associatedInstruction
```

## Incremental Sketch

- Events need to have:
    - A global timestamp
    - Metadata that is specific to the event type
    - The ability to reference the tags of specific predecessor events
    - The ability to end an event chain by recycling the tags of completed events
- In increasing order of complexity:
    - Tagless single-trigger event with metadata
    - Two tagless single-trigger events that have to be distinguished (this distinguishing can be done by the metadata alone, I think)
    - Two tagged events where the second event consumes the tag produced by the first

### Single-Trigger Tagless Events

```scala
// manual implementation of the simplest type of event (tagless single-trigger event with metadata)
class Example extends Module {
    val io = new Bundle {
        val in = UInt(32.W) // just a random signal that's used as the metadata for the event
    }
    // Global cycle counter (ideally only created once per DUT, but we can allow creating one per event for now)
    val cycleCounter = RegInit(0.U(64.W))
    cycleCounter := cycleCounter + 1.U

    val (counter_val, counter_wrap) = Counter(true.B, 100)
    when (counter_wrap) {
        chisel3.printf(cf"Cycle: ${cycleCounter}, Event: CounterWrapEvent, Metadata: ${io.in}")
    }
}
```

```scala
// generalizing the above example
def event[T <: Data](trigger: Bool, name: String, metadata: T): Unit = {
    val cycleCounter = RegInit(0.U(64.W))
    cycleCounter := cycleCounter + 1.U
    when (trigger) {
        chisel3.printf(cf"Cycle: ${cycleCounter}, Event: ${name}, Metadata: ${metadata})")
    }
}

// consider 2 tagless single-trigger events that can be distinguished by event type and metadata
class Example extends Module {
    val io = new Bundle {
        val in1 = UInt(32.W)
        val in2 = UInt(16.W)
    }
    val (counter_val1, counter_wrap1) = Counter(true.B, 100)
    val (counter_val2, counter_wrap2) = Counter(true.B, 10)
    event(counter_wrap1, "CounterWrapEvent1", io.in1)
    event(counter_wrap2, "CounterWrapEvent2", io.in2)
}
```

### Events with Tags

```scala
class Cpu extends Module {
    // Trigger events from RTL
    val newInstFetched: Bool = ???
    val l1Access: Bool = ???
    case class FetchMetadata extends Bundle {
        val inst = UInt(32.W)
        val pc = UInt(64.W)
    }
    case class L1RequestMetadata extends Bundle {
        addr: UInt
        accessType: Read | Write
    }
    case class L1ResponseMetadata extends Bundle {
        data: UInt
    }

    val (fetchTag, fetchTagRecycleEn, fetchTagRecycle) = event(cond=newInstFetched, "Fetch", new FetchMetadata.Lit(_.inst -> ???, _.pc -> ???))
    val fetchInFlight = Reg
    event(cond=l1Access, "L1Access", new L1RequestMetadata(_.addr -> ???, _.accessType -> ???), parentTag =
}
```
