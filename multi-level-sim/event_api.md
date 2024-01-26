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

- Events need to have:
    - A global timestamp
    - Metadata that is specific to the event type
    - The ability to reference the tags of specific predecessor events
    - The ability to end an event chain by recycling the tags of completed events
- In increasing order of complexity:
    - Tagless single-trigger event with metadata
    - Two tagless single-trigger events that have to be distinguished
    - Two tagged events where the second event consumes the tag produced by the first

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

```

