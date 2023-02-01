# Chisel Sequences

## 2/1/2023

- Retreat is May 22-24 (Mon-Wed), 2023
- ScalaSeq -> PSL formula
    - WIP: PSL formula -> HOA (by calling SPOT) (use sequences.backend.Spot.callSpot)
    - WIP: interpret the HOA over a concrete Scala trace (and compare it to our manual implementation of assert/cover of Scala sequences)
- Testing more sequences types for the SpotBackend (likely many of them aren't being handled properly)
    - There is a bug! PSL concat of 3 or more APs doesn't yield the correct FSM.
    - Investigate the correct encoding of multiple AP concats
    - Incorrect way: a & (X b & (X c))
    - Correct way (but ugly): a & (X b) & (X X c)
    - Click help here (https://spot.lre.epita.fr/app/) and look at the section "PSL Infix Syntax"
    - Investigate the concatenation and fusion operators
- Another Spot test line 71 - fails for no good reason
    - Suspect that the PSL emission of `SeqConcat(a, SeqOr(b, notC))` is wrong
    - Try unit testing this, and manually checking the FSM using Spot's webapp
- Just do something like this to get around the sequence wrapped in a Property
```scala
case SeqImplies(s1, p1) =>
val s = p1 match {
  PropSeq(s) => s
}
s"(${sequenceToPSL(s1)} |->${s})"
```

## 1/25/2023

- Send you details about next retreat (late May)
- tapeout, 140, 130/189, 152, EE 219C (formal methods)

### Vighnesh's TODOs

- Rebase on top of Kevin's upstream
- Create a dev branch on my repo (and you can merge your stuff on that dev branch)
- Unify the testing infrastructure for SPOT and SequenceFSM backends

### TODOs

- Check on the Scala sequence to SPOT convertion + HOA parsing + assertions

- Write sequences for Saturn L1 cache identical to the ones currently written in SystemVerilog
- `SpotBackendTests.scala` write some tests that exercise all the potential PSL properties that we can emit from Chisel
    - Goal: verify that the automata generated from the PSL -> SPOT -> HOA -> Chisel circuit MATCHES the interpretation of the raw HOA automata in Scala
    - Tasks:
        - Continue work on scala sequences - to convert ScalaSeq -> Spot PSL string -> HOA
        - Validate our HOA processing engine `def process(h: HOA, apTrace: Seq[Set(Int)]: (Bool, Option[Int])` (golden model for what the Chisel circuit should do)
        - Extend tests in `SpotBackendTests.scala` to cover more types of properties
            - So far, only `SeqConcat`
            - Can we also test `SeqFuse`, `SeqOr`, `SeqAnd`, multiple chained `SeqConcat`?
    - Eventual flow:
        - Generate a bunch of properties in the Sequence ADT
        - From each property: generate its HOA automata and Chisel monitor circuit
        - Fuzz both the automata and the Chisel monitor circuit to check for equivalence
    - Later:
        - Formally prove equivalence between a SPOT HOA and a Chisel circuit representation
