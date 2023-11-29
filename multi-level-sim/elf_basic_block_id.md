### Observations
- Not all "section" starts are targets, but that is because the program doesn't call many functions (from static libs) even if the binary contains them
  - Can be addressed by link time optimizations
  - Each section start is a function

### Merging in
- [ ] Tests
- [x] Top-level argparse
  - [x] log level
  - [x] filename
- [x] tqdm 
- [x] mypy
- [x] Code cleanup
  - [x] Refer to cf instrs only with pcs
  - [x] Common code 
  - [x] Lists
    - c.jal = jal x1, offset[11:1] -> target only
    - c.jalr = jalr x1,rs1,0 -> no target
    - c.j = jal x0,offset[11:1] -> target only
    - c.jr = jalr x0,rs1,0 -> no target
    - jal offset and jal rs1, offset are both valid!
    - call offset and tail offset are also valid pseudoinstructions

### Improvements
- [x] instr bits
  - [x] capture
  - [x] use for prev/next instr
  - [x] better interval tree
    - [x] addInterval
    - [x] getInterval
- [x] Use annotations
  - Not really, good for debugging but probably not a target in current file
- [x] Name passes/stages
- [x] Update list of cf instrs

### Ghidra comparison
- Looks like Ghidra uses decompilation here, so some differences should be expected
  - We don't decompile
  - Decompilation is generally imperfect
- [x] Ghidra ~~misinterprets sext.w instrs (rv64 c) as c.jal instrs (rv32 c)~~ matches
  - 10 occurences in hello-world
  - Consider by hand?
    - Ending basic block after `8000055c` is due to a target at the next instr
    - Not wrong! Both implementations match
- [ ] Ghidra inlines jal-ret pairs ~~that are a single basic block called just once~~ for as yet unknown reasons
  - ~~[ ] Write coalesce pass~~ 
    - this will need graph analysis because it can be nested
    - not worth it
  - Doesn't matter for us because it'll be caught by dim reduction
  - [x] nvm Ghidra does it wrong => `__init_tls` -> `memcpy` (also target of others + has divergence within)
  - [x] Is it just doing this for all jals? Find a jal where ghidra ends the basic block
    - Nope. It deifnitely breaks blocks on some jals, see `80000396`
    - Just don't know why/how it decides
      - And no, we don't have the same "next instr is a br/j target" issue here
- [x] Ghidra inlines jalrs
  - [x] Check other jalrs - 22 in all
  - Differing treatment - 1, 2, 4-15, 17-19, 21, 22
  - Similar treatment - 3 (`800009d6`; next instr is a jump target), 16 (`80001b46`; next instr is a branch target), 20 (`80001cf6`; next instr is a jump target)
- [ ] Our implementation handled jr differently from ghidra at `800015fa`
  - Ghidra breaks before entrance, and another after jr
  - We only break after jr
  - Can't find jump target for instr right before
  - [ ] Check jump table?
- [ ] Ghidra finds a jump target at `80001722`, we don't
  - [ ] Check jump table?

### clang comparison
  - [ ] Build libgloss with clang
  - [ ] Build hello-world with clang
