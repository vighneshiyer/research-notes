# Coverage APIs

## Overview

We have 3 projects related to coverage APIs for Chisel and their usage in chiseltest:

• Web frontend for coverage visualization and analysis (Grace) (NOTE: notes for the coverage frontend are in a separate file now)
• Integration of simulator independent coverage coverpoint passes (FIRRTL) into chiseltest (https://github.com/ekiwi/simulator-independent-coverage)
    ◦ line, toggle, cond, custom cover (ready && valid - transaction, valid && !ready - backpressure, !valid && ready - consumer is ready)
    ◦ subexpression coverage
    ◦ cross register toggle coverage (a pseudo-bughunting metric)
    ◦ mux toggle coverage
• Infrastructure for coverage wrangling
    ◦ All simulators should be able to write to a common coverage database format - UCIS (https://www.accellera.org/activities/working-groups/ucis)
    ◦ Alternative simulator integration (iverilog, FPGA - FireSim, treadle, Java ESSENT) of coverage passes

## 11/10/2022

• Yi: script to convert a coverage.dat into a coverage IR into a UCIS database
    ◦ Whatever coverage IR we design should support a superset of UCIS constructs and we should be able to parse the VCS UCIS database as well (pyucis doesn’t support structural coverage databases yet - maybe never)
    ◦ Next: parse coverage.dat file into the Same internal representation as the VCS UCIS database
        ▪ Serialize the IR in some format (e.g. json)
    ◦ Next: Create a very simple Verilog RTL module and testbench that exercises the coverage we care about (branch, toggle, line, conditional)
        ▪ Collect coverage from VCS and Verilator using all the flags available
        ▪ Make sure we can parse all those formats
            • have a manually defined reference for the coverage counts
        ▪ FSM coverage from VCS
        ▪ Nested modules
        ▪ Per-instance coverage collection
    ◦ Long term vision: we want a database of a bunch of UCIS and coverage.dat files that we can use to validate our parsers

## 11/4/2022

• Run the 151 labs (whichever one you want) using VCS with coverage collection enabled (for line, toggle, branch, conditional, fsm, …)
    ◦ Experiment with all the flags that VCS has to offer
    ◦ Just so that the coverage database contains data that reflect the coverage collected by each flag
    ◦ -> you may want to write a pure Verilog simple design and testbench for which you know what the coverage counts for each type of coverage ought to be
    ◦ Use urg to dump an HTML report, just to take a look
• Use PyUCIS to parse these databases and investigate the UCIS format itself, so you understand it (https://github.com/fvutils/pyucis)
    ◦ Use the ‘ucis’ command line program itself at first
    ◦ Then try programmatically going through the database using the Python API
    ◦ Reference the UCIS specification against the XML databases emitted by VCS
        ▪ https://www.accellera.org/images/downloads/standards/ucis/UCIS_Version_1.0_Final_June-2012.pdf
        ▪ https://fvutils.github.io/pyucis/reference/xml_interchange.html
• Long term: write a program that can convert the Verilator coverage.dat file(s) into a UCIS database
    ◦ Initially, we can just use PyUCIS to create a UCIS database from parsing the Verilator coverage.dat file
        ▪ Use urg (VCS tool) to convert the database into an HTML visualization
    ◦ Later, when performance becomes important, we can rewrite this

## 11/3/2022

Yi:
    • The LineCoveragePassTest fails right now, we need to figure out why
        ◦ Start off with a simple DUT that you have full understanding of and write a test for it, then use the expected values of line coverage to check against what the simulator result actually is (and the parsing function too)

## 10/27/2022

Yi:
    • Line coverage pass: https://github.com/ekiwi/simulator-independent-coverage/blob/main/coverage/src/coverage/LineCoveragePass.scala
        ◦ We want to just copy this over almost ‘verbatim’ to chiseltest in the line-coverage branch
        ◦ We also want to copy over the test for this pass
            ▪ https://github.com/ekiwi/simulator-independent-coverage/blob/main/coverage/test/coverage/LineCoverageTest.scala
        ◦ Verify the test works
        ◦ Confirm your own understanding of the line coverage pass

val c = Chisel circuit
test(c()).withAnnotations(Seq(LineCoveragePass, …)) { dut => … test }
Verilator emits a coverage.dat
visualize it

## 10/20/2022

• Yi: https://github.com/pidan-bilibili/Chisel-ALU
    ◦ VCS coverage collection in 151 makefile
    ◦ chiseltest clone and test is OK
    ◦ firrtl clone and test has various spurious issues (related to destroyed processes)
        ▪ Install yosys and try again (brew install yosys)
    ◦ Vighnesh: add a branch to chiseltest with the AnalyzeCircuit example that we can play with (DONE)
        ▪ https://github.com/ucb-bar/chiseltest/tree/line-coverage
        ▪ testOnly chiseltest.coverage.AnalyzeCircuitSpec
        ▪ I have also added an example of how to return an annotation from a pass and inspect it (see the test AnalyzeCircuitSpec)
    ◦ TODO: work on line coverage pass integration and test (copy over from simulator-independent-coverage and test inside chiseltest, and maybe write a few more tests + understand how the line coverage pass actually works)
    ◦ Vighnesh: clean up the chisel-bootcamp and firrtl examples to use the latest API

## 10/13/2022

• Yi
    ◦ Also try to implement the FIFO in Chisel
    ◦ VCS coverage collection (use the eda1.eecs.berkeley.edu machines, get an account here https://acropolis.cs.berkeley.edu/~account/webacct/)
        ▪ cov_ug: https://drive.google.com/file/d/1n93_sMzo2jGFMunrmDiclAmvvLjPRTYP/view?usp=sharing
        ▪ vcsmx_ug: https://drive.google.com/file/d/1C0nyWGc8-xqo2rLlJuwArOXgD-c0EexH/view?usp=sharing
        ▪ -cm line (see vcs_ug.pdf)
        ▪ urg -dir <UCIS dir> -> dump HTML
        ▪ Look at the HTML
        ▪ TODO: do this for 151 labs, find your own coverage holes (line, toggle, branch, condition, fsm, …)
    ◦ Compile and test chiseltest and firrtl locally
        ▪ + try chiseltest Verilator coverage collection: https://github.com/ucb-bar/chiseltest/blob/main/src/test/scala/chiseltest/backends/verilator/VerilatorCoverageTests.scala
    ◦ Implement the firrtl tutorial pass inside the firrtl repo and verify it works with a unit test

## 10/3/2022

• Vighnesh
    ◦ Add everyone to the BAR slack (pending invites)

◦ Writing RTL should be familiar to everyone in a few weeks
◦ Send the alu.v and alu.cpp files so you can reproduce coverage reports
    ▪ I have posted the files on Slack (you will get an invite shortly)
    ▪ Install verilator on your laptop
    ▪ Run: verilator --cc alu.cpp alu.v --build --coverage-line --exe
    ▪ Run: ./obj_dir/Valu
    ▪ Inspect: coverage.dat
    ▪ Play: See the commented ‘cover’ statements in alu.v? These are custom user-defined coverpoints. Try to enable some of them and write your own - any boolean expression inside cover() will be counted by the simulator and emitted to the coverage.dat file. You will need to use “--coverage-user” when invoking Verilator to capture user-defined cover points.

◦ Make sure you can run the Verilog testbench using iverilog too
    ▪ Uncomment the alu_tb in alu.v
    ▪ Run: iverilog -g2012 alu.v
    ▪ Run: ./a.out
    ▪ Play with the testbench so you understand it

◦ Collect verilator coverage from chiseltest
    ▪ You have to pass in a custom flag into the test() function so chiseltest uses Verilator as a backend (by default it uses treadle, which is a pure-Scala RTL simulator)
    ▪ test(new ALU()).withAnnotations(Seq(VerilatorBackendAnnotation)) { c => // your test logic }
        • See: https://github.com/ucb-bar/chiseltest/blob/main/src/test/scala/chiseltest/backends/verilator/VerilatorBasicTests.scala
    ▪ Here is how you also enable coverage collection from chiseltest (https://github.com/ucb-bar/chiseltest/blob/main/src/test/scala/chiseltest/backends/verilator/VerilatorCoverageTests.scala)
        • See the generated coverage.dat file in the test_run_dir corresponding to your test name

• Yi
    ◦ Reproduce Verilator example with coverage collection (produce coverage.dat file)
        ▪ see if we can use lcov to visualize coverage (convert coverage.dat into HTML report with verilator_coverage) (https://github.com/linux-test-project/lcov)
        ▪ verilator_coverage -write-info
        ▪ lcov –generate-html ‘’
    ◦ Collect coverage on 151 labs with VCS (see cov_ug.pdf I emailed you) - you will have to modify the Makefile slightly to add a flag to the VCS invocation
    ◦ Go through Ch 4 of the bootcamp again more closely, actually modify some code and understand how it works (https://github.com/freechipsproject/chisel-bootcamp/blob/master/4.4_firrtl_add_ops_per_module.ipynb)
    ◦ Compile and run tests for chiseltest on your laptop (https://github.com/ucb-bar/chiseltest)
    ◦ Compile and test firrtl locally (https://github.com/chipsalliance/firrtl/)
        ▪ Implement the tutorial pass in the firrtl repo and write a unit test for it
