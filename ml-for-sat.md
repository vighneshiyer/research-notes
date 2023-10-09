# ML for SAT/SMT/Verification

## Technique

- https://github.com/arashardakani/HWverification
- ??? a mystery to me

## Applications

- Solving SAT formulas
    - Represent the SAT formula as a DAG from the free variables (inputs) to the output (true / false)
    - **Evaluation**
        - Compare the ML approach against regular off-the-shelf SAT solvers in time to solve a given problem
        - One issue is that we can never say UNSAT for sure with the ML approach - need a reasonable timeout after which we claim 'likely UNSAT'
- Sampling a SMT formula for legal models (uniformly sampling a constraint space)
    - We would hope the ML approach produces more uniform sampling vs other techniques
    - **Prior Work**
        - Guidedsampler: https://ieeexplore.ieee.org/abstract/document/8894251
        - SMTSampler: https://ieeexplore.ieee.org/abstract/document/8587695
    - **Evaluation**
        - Compare the ML approach against prior work (GuidedSampler/SMTSampler) for sampling SMT formulas
        - Also compare it against SystemVerilog-based commercial simulator constrained samplers
        - Also compare it against PyVSC
- Bounded model checking (BMC)
    - Unroll a transition system for N iterations -> turn into SMT -> solve as usual
- Automatic test pattern generation (ATPG)
    - Construct a sequence of inputs such that we see all the possible internal toggle patterns in a circuit

## TODO

### SAT

- [x] Get SAT competition benchmarks: https://satcompetition.github.io/2023/downloads.html
    - Are the formulas in CNF?
- [x] Get a bunch of pigeonhole problems in CNF form
- [ ] Download the bit-blasted AIGER benchmarks: https://fmv.jku.at/aiger/
- [ ] Try to parse the formulas in Python
- [ ] Figure out how to programmatically construct a Pytorch model that represents the formula

### SMT

- [ ] First formulate a word-level representation of circuits as ML models
- [ ] Fetch the SMT benchmarks for QF_BV or QF_LIA (place them on `a` machines)
    - https://smt-comp.github.io/2023/benchmarks.html
- [ ] Make sure we can parse a benchmark via [pysmt](https://github.com/pysmt/pysmt)
- [ ] Investigate whether `pysmt` can bitblast the SMT statements into only boolean signals
- [ ] Dump a simple SMT circuit
- Later:
    - [ ] Convert the SMT circuit to a Pytorch model
