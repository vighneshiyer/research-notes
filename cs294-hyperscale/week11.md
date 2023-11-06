## Q100: Architecture and Design of a Database Processing Unit

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Relational databases are implemented using pure software running on CPUs utilizing SIMD instructions to the extent possible.
They operate by processing a SQL query, constructing a query plan, and executing that plan on tables and columns.
Database query processing is based on streaming data through functions that perform some kind of transform/filtering/joining (a fixed set of primitives).

#### Summarize the paper's contributions. length: one paragraph.

The authors propose an ISA and architecture specialized for database query execution.
Their architecture consists of a spatial array of PEs onto which spatial instructions are mapped - their compiler (they don't have a real compiler, they manually translate queries) breaks query execution into a sequence of temporal instructions that utilize a portion of the Q100 resources.
They perform a PPA-based DSE of different mixes of PE types and evaluate cross-tile communication bandwidth requirements.

#### Summarize the weaknesses of the paper. length: one paragraph.

How is pipeline balancing done? Certainly some operators have much higher throughput than others - what is the utilization? If we ever have to use the CPU, then the bandwidths will defintely not be matched and throughput will suffer massively.
Aren't databases memory bound in the first place? Does it really make sense to have a compute accelerator for database queries?
SQL is a huge language - there is no chance that any accelerator could implement all the things used in typical queries - then delegating to the CPU will defeat the fine-grained parallelism compute benefits of acceleration.
Where did their SQL queries come from? Are they representative of real workloads? TPC-H seems like quite a synthetic benchmark suite.

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

First, this technique need to be examined in the context of real SQL queries.
Take a dataset from profiling a real application and statically evaluate whether the existing set of PE tiles is sufficient to execute the queries without spilling compute to the CPU.
Then develop a compiler to evaluate the feasibility of mapping queries to the accelerator fabric.

## Cores That Don't Count

### Review

#### Summarize the existing "state of the world" as the paper describes it (intro, motivation, related work). length: one paragraph.

Non-soft-error related sporadic core failures are observed in datacenters today where computations silently return the wrong result.
This is very bad - these CPUs pass manufacturing tests and they randomly begin to fail in silent and non-obvious ways.
Google observes "a few" mercurial cores per several 1000 machines - even though that is a low ratio, it is still significant.
Defects associated with aging were typically "fail-stop" or "fail-noisy", but we observe that now defects are exposed for a particular core, for only very specific instructions.

#### Summarize the paper's contributions. length: one paragraph.

The authors describe the impacts of these silent errors in Google's datacenter. They describe tests they developed to uncover silent defects in CPUs.
The authors discuss how mercurial cores produce errors that are difficult to detect/correct using existing mechanisms (ECC, CRC).

#### Summarize the weaknesses of the paper. length: one paragraph.

The authors don't disclose anything important. They also don't discuss the fault model for CEEs and how it contrasts with understood fault models for soft errors (due to radiation) or for aging related defects or manufacturing defects (which are well understood).

#### Describe a hypothetical class project you could work on that builds on the insights of this paper/addresses weaknesses/etc. assume you have access to similar resources as the paper authors. length: one paragraph.

Release a detailed report on the observations made about mercurial cores (how they exhibit errors, how those errors are eventually detected, which instructions fail, what are the bad components of the CPU).
Evaluate Intel's and AMD's testsuites (e.g. Intel's opendcdiag) on their ability to find mercurial cores.
Perform a quantitative cost-benefit analysis on spending CPU cycles detecting mercurial cores vs using more cycles for application code.
