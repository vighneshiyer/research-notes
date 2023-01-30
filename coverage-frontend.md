# Coverage Frontend

## 1/25/2023, Wed

- Sungwoong's interests from the retreat: GPU RTL design project, Chipyard infra + whole integration story
- Taking: 151 (fpga), 152, 61b

### Vighnesh's TODOs

- TODO: give you a more complex Verilator .dat file we can look at

### TODOs

- https://github.com/vighneshiyer/hw-coverage-viewer
    - Pull this and try to get the local HTTP dev server working + the tests for the coverage parser
    - All sources in `hw-coverage-viewer/src/cov`
    - Run tests with `npm run test:unit`
- Get familiar with the tools
    - vue.js (https://vuejs.org/tutorial/)
    - typescript (https://www.typescriptlang.org/docs/handbook/intro.html) (https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-oop.html)
- Add assertions to `parse_line` unit test
- Test the `parse_data` function
- Add type annotations to `parser.ts`

### Initial Target

- Parse Verilator .dat files and display coverage info
    - Start with the ALU simulation .dat file
- Develop a simple Vue component that can take raw coverage data in the coverage struct format (`CoverageEntry`) defined in `parser.ts` and the source code of the Verilog file, and display the source code annotated with line numbers and coverage counts
    - Inputs: parsed coverage.dat file, a string representation of a Verilog file
    - Display: a HTML view of the Verilog source with coverage counts annotated on the left hand side
