# Hardware IR

## Issues We Have Now

So we have FIRRTL. There is a FIRRTL ingestor in circt
HW dialect in circt


In-memory representation
Incrementalism
Dedup natively
Loss of semantics
Unified pass writing mechanism (as rewrite rules ideally)
Native mixed abstraction
Module abstraction is unnecessary and restrictive (port-oriented circuits are a problem for reparenting and other hierarchy manipulation / grouping)
