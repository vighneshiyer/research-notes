# Chiseltest FFI Performance

## 5/3/2023

- All VerilatorComplianceTests pass locally, TODO: test on another computer and validate via CI

### TODOs

- Add a switch to go between JNI and JNA, via some annotation
    - See: https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/simulator/VerilatorSimulator.scala#L17 for how to add options to the Verilator invocation via annotations
    - Here's how you unpack the value from an annotation: https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/simulator/VerilatorSimulator.scala#L268
    - Here's how to add annotations when invoking the chiseltest simulator: https://github.com/ucb-bar/chiseltest/blob/main/src/test/scala/chiseltest/backends/verilator/VerilatorCoverageTests.scala
- Remove duplication between JNIUtils and JNAUtils
- Do a final performance benchmark with the GCD test
- Open a draft PR on chiseltest, note the above things are still TODOs

## 4/19/2023

Updates:
- Able to compile bridge library from source from Scala, thanks for the help and code cleaning
- Started on looking into getting the verilator so interface to work through jni bridge library as opposed to jna native access
    - looking at JNASimulatorContext --> need to create a similar one where parameters are slightly different because no longer need the actual so, but just the so_id which can then be passed in as a parameter to JniAPI native calls which go to the bridge library
    - compileAndLoadJNAClass --> compileAndLoadJNIClass which will laod the so from the path and initialize via `sim_init`, have return type be a new interface which can then be the parameter that get passed into JNISimulatorContext
    - in VerilatorSimulator.scala we want to compileAndLoadJNIClass as opposed to jna in the generate context methods
- option 1: create a new TesterSharedLibInterface that will delegate calls to the bridge library ( would require taking in the so_id)
- option 2: don't even create a new interface just a collection of (soId, sPtr) from compileAndLoadJNIClass that we directly pass into the simulator context and the simulator context will just call the bridge library directly rather than delegating the need to .getFunction for JNA, since loading the so in compileAndLoadJNIClass will have already done that for us (avoids the layer of indirection/abstract)

- To fix:
    - Use longs when appropriate in the bridge library vs ints
    - set_args needs to take a Array[String] - figure out how to do this in JNI
    - Later: need a switch between JNI and JNA in VerilatorBackend

```text
long - JVM primitive
Long - boxed JVM object (wrapper around long that adds OOP functionality)

Long - Scala long (can either be a JVM primitive or a boxed JVM object based on how it is used)
```

## 4/12/2023

- Got a basic unit test of the bridge library working in the chiseltest repo
- We want to compile the bridge library from source from Scala (maybe use jni-utils?)
    - You have access to this dependency: https://github.com/com-lihaoyi/os-lib
    - https://github.com/com-lihaoyi/os-lib#os-proc-call
    - Create a temp dir: https://gist.github.com/amr/cc51a04d3511d272db36 to put the compiled bridge library in
    - One challenge is to get this working in multithreaded unittest environment (defer this for now - we might need a specific type of filesystem that support directory level locking)
        - Need some way to lock the dir while the .so is being compiled

## 4/5/2023, Wed

- Can now call .so functions through the bridge library (in Scala!) without any errors!
    - This is with the Foo passthrough module and it indeed works as expected
- You know everything now do to the chiseltest integration
    - Need to compile the bridge library through JNI when integrated into a library
- First objective:
    - Inside chiseltest, add your bridge library C code
    - And make sure you can compile it using whatever JNI stuff (`javah`, etc.)
    - Then load the library in a chiseltest unit test, and invoke some functions from there (load an .so e.g. VFoo, and invoke it)
    - Fork chiseltest under your username and do the jni work in a branch
    - Add `fork in test := true` in the chiseltest `build.sbt`
- Investigation on how to distribute JNI libraries such that users who depend on chiseltest can compile the JNI .so on their own machine (different architecture, different OSs)

## 3/22/2023, Wed

- sim_init returns a void* which needs to be passed back to Java as a jlong and then passed back to the testbench API functions (e.g. step, poke). Some JNI header file changes are necessary

## 3/8/2023, Wed

- Something broke in ffi_benchmarks wrt running `javah` with the modified bridge library, but a fresh clone of ffi_benchmarks fixed the problem (there were some latent build artifacts breaking something, but we don't know what)
    - The problem was running `sbt javah` terminates with a `nio.file.ProviderNotFoundException` for no obvious reason - the stack trace also doesn't reveal anything. The JNI plugin may have messed something up that is persistent even after removing it as a dependency.
    - Anyways, Oliver is just going to manually copy over the bridge library mods to the fresh clone and keep working

## 3/1/2023, Wed

- Run the Verilator tests using sbt like: `sbt test chiseltest.simulator.VerilatorBasicCompliance`
- You will get the test artifacts here:
    - `/home/vighnesh/80-temp/chiseltest/test_run_dir/verilator_should_be_able_to_load_and_execute_a_simple_combinatorial_inverter_generated_from_Chisel`
    - Inside the `verilated` folder you have `VFoo` which should be a regular .so that you can load using your bridge library
    - Its API should look like `Foo-harness.cpp` and see the EXPORT'ed functions
- Thing to try: load this .so using your JNI bridge library and invoke its functions from Scala
    - This will require modifying your bridge library to have the same API as the harness
    - Invoke the functions according to what you would actually do in the chiseltest backend
        - ptr = sim_init()
        - poke(??, ??)
        - update() // force reevaluation of the DUT (for combinational paths)
        - value = peek(??, ??)
        - assert(value == ???)

## 2/22/2023, Wed

- Bridge library is about 2x slower than the native JNI access, still room for further improvement by moving dlerror() check to load_so function
- This is already good enough to begin chiseltest implementation
- First steps:
    - Right now the Verilated C++ code is compiled with a generated harness into a regular .so which is loaded with JNA
    - Add the code for the bridge library that has an interface with the same functions in the harness
    - Inside VerilatorSimulator, we want to take a different path than the JNA one. We create a regular .so as usual with the Verilated C++ and harness file. But instead of loading that .so thru JNA, we load our JNI bridge library, and then load that .so through the bridge.
    - This will require another JNISimulatorContext class.
- VerilatorBasicompliance actually builds an .so, but it doesn't have a .so path suffix; but it indeed is designed to be invoked via JNA
- We will need to write our own harness generator and build function that builds a regular .so for invocation from the C bridge library

## 2/15/2023, Wed

Updates:
- Cached function pointer in load_so rather than recalling `dlsym` each time the API was called
- observed around ~5x speed up from previous benchmarks
    - overall about ~2x slower than a direct JNI function invocation
- Started reading into JNAHarnessGenerator
- compiled most recent benchmarks: https://docs.google.com/spreadsheets/d/19nesLGxOIy0zceCflExHISJboLFcFRJ1nLDIjGdui8I/edit#gid=0
    - only two benchmarks really: pre and post caching function pointer

Questions:
- Why are there two code buffers? Don't they just get concatenated together in the cpp file?
    - Is it for scope/visibility purposes?
- General logic behind harness generator:
    - function definitions and shared object compiling and linking all defined in JNAUtils, which will invoke the native implementation defined in the harness generator
        - JNAUtils handles a lot of the setup whereas HarnessGen is for the native impl
    - `struct sim_state`: represents one instance of a circuit simulation?
    - assigns ids to each input/output which are passed into as a argument to each API call
    - harness generator defines codeBuffer which is essentially inline C++ code
    - harness has native implementation of each API function (peek, poke, peekwide, pokewide, step, coverageAPIs, etc.)
    - harness has private helper functions for ????
        - commonCodeGen vs. codeGen

## 2/15/2023, Wed

Updates:

- Tried to print out loaded JNI libraries, ran into some illegal reflective access issue
    - Couldn't seem to convert ClassLoader value into sequence of strings
- Benchmarking progress:
    - located in `jni_api_test_benchmark`
    - first is just calling the call_add_one function repeatly, not including the load_so (calling bridge native library which is then loading another .so and calling it through dlsym)
    - second is directly calling DummyAPI.add_one repeating (directly calling native library)

- Very interesting result: using the bridge library over JNI to invoke an underlying so vs invoking the so via JNI directly actually causes 7-10x performance degradation!
    - Suspicion: this is caused by calls to dlsym on every invokcation of add_one through the bridge library
    - We can avoid this by caching the function pointer returned by dlsym
- TODO:
    - Every time you load a new so, you should also fill in caches for all the functions that could be invoked on that so
    - You can just maintain an array for every function
    - We only have about 6-10 functions for chiseltest, so this is OK, we can manually duplicate code - the important thing is to avoid overheads
- Enumerate the results of all your benchmarks in one table we can reference later
    - Include the failed / unusually slow attempts (just like this one, we want to know how much caching helps us)
- Skim thru the code in this folder: https://github.com/ucb-bar/chiseltest/tree/main/src/main/scala/chiseltest/simulator/jna
    - Investigate the API that your bridge library needs to support
    - Template is here for JNA invocation of Verilator C++ code: https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/simulator/jna/VerilatorCppJNAHarnessGenerator.scala

## 2/7/2023, Wed

- Problem: if we load the bridge .so once with System.load, on the second sbt test invocation, we skip the bridge .so load and directly attempt to invoke its functions, and that gives a Link error indicating that the library may no longer actually be loaded
- TODO:
    - Can you cache the pointer to the native function? **NO** this will not work.
    - Explore some of the SO answers on figuring out what native libraries are loaded in a JVM. Likely the jnibridge library is not loaded in the second run.
    - https://github.com/sbt/sbt-jni (use their flow and directory structure to define native packages and use their annotation to load the bridge so without using System.load directly - which seems to not work nicely with a long running sbt session on a single JVM)
    - Try using `mill` as the build system (https://github.com/com-lihaoyi/mill). The build file for mill is `build.sc`.
        - Install mill: https://com-lihaoyi.github.io/mill/mill/Installation.html#_downloading_mill
    - Try using `fork in run := true` in `build.sbt` **"Resolves"** the problem, but have to load the bridge so every time (that's ok).
- Next: use JMH to benchmark bridge vs direct library FFI.

Updates:

- Successfully added system property to only load when System.getProperty("jnibridge.loaded") = "true"
- Running into new issue: `UnsatisfiedLinkError` on second call to loadSo (no longer classloader loading native object twice)
- The name exported by your service program must match the name expected by the JVM; otherwise, one of the call-time (as opposed to load-time) UnsatisfiedLinkError is thrown.
- https://groups.google.com/g/simple-build-tool/c/KgOAs4fbGBo/m/-dOYOpWDYtEJ
  - one person seemed to resolve this, but can't seem to figure out solution
- Different attempts:
  - add native so into unmanaged Jars in sbt build file so loads with sbt rather than statically
  - looking into sbt-jni plug-in and NativeLoad (may add additional overhead)

## 2/1/2023, Wed

- To avoid loading the same .so twice with System.load, we can set a global JVM-wide system property upon the first load, and then check whether that property has been set to avoid a second load
    - https://docs.oracle.com/javase/6/docs/api/java/lang/System.html#setProperty(java.lang.String,%20java.lang.String)
    - setProperty (JVM-wide property)
- TODO: benchmark the overhead of the .so proxy library (libproxy) vs calling the .so we care about (temp_lib) directly
- Summer retreat: May 22-24 (Mon-Wed), 2023 (Napa, CA)

```scala
class Thing {
  def method: String
}
object Thing {
  def staticFn: Int
}
Thing.staticFn
Thing().method()
```

## 1/25/2023, Wed

- Classes: tapeout, 152, 105
- Vighnesh's TODO: send Oliver the slides from the retreat
- https://github.com/oliveryu11/ffi_benchmarks

## 11/30/2022

- The function that is called from the JVM to load an .so may be called by multiple threads
    - We need to synchronize access to the id counter
    - We don’t want to lock read access to the array of .so pointers, so we should preallocate a large (1024) element array to store these pointers - if we try to load too many .so’s then just blow up
- 2 or more .so’s with the same API
- bridge .so which should contain the load_so function and all the same APIs as those above .so’s
    - Invoke load_so from the JVM, get back an id
    - The JVM uses that id to call the bridge .so functions which proxies to the underlying .so

## 11/16/2022

- TODO Vighnesh: verify with Kevin wrt what the issue is with using JNI directly to load multiple .so’s with the same interface
    - https://github.com/ucb-bar/chiseltest/pull/349#issuecomment-891506091
    - The issue is that JNI requires a Java class per .so. One solution is (at test runtime) to generate a Java class as Java code, compile it using javac, and inject it in the current classloader
    - That last part is tricky and flaky, so this is why a bridge library might be a better idea
- TODO: work on the JNI .so loader bridge library, delegate calls to underlying .so’s based on their id’s and assume that all the underlying .so’s have the same API surface

## 11/2/2022

- https://github.com/ucb-bar/chiseltest/tree/main/src/main/scala/chiseltest/simulator
    - Implement a jni package with a similar interface to jna
- TODO: Implement a C library that can be called from Java to load .so’s with the same interface, but potentially different states
    - `int loadSo(string so_name)`
    - `int call_add_one(int so_id, int argument)`

## 10/19/2022

- Get the example working locally
- Profile using JMH (or any other profiling technique) - benchmark the time to call this function
- JNA direct call interface
    - Also investigate this: https://github.com/java-native-access/jna/blob/master/www/DirectMapping.md and its impact on performance
- Try using JNI for the same call and benchmark it

## 10/13/2022

- https://github.com/oliveryu11/jna_test
- https://github.com/java-native-access/jna#using-the-library (look at these docs)
- Try to import the custom .so using JNAUtils.scala as a reference (from chiseltest)
    - Also in JNAUtils, the flags for compiling a custom .so are provided (in ccFlags and ldFlags)
    - https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/simulator/jna/JNAUtils.scala
- Also investigate doing the same thing with JNI, how difficult it is to use vs JNA (and what is the performance overhead for a very simple function (e.g. add_one) where we are measuring the FFI overhead purely)
    - Also investigate this: https://github.com/java-native-access/jna/blob/master/www/DirectMapping.md and its impact on performance
- `test(new ALU()).withAnnotations(Seq(VerilatorBackendAnnotation)) { c => // your test logic }`
    - See: https://github.com/ucb-bar/chiseltest/blob/main/src/test/scala/chiseltest/backends/verilator/VerilatorBasicTests.scala

## 10/5/2022

- JNA hacking in progress, working on calling external C library
    - Finish chisel-bootcamp (including chapter 4 since it will be helpful when digging through chiseltest code)
    - Set up the chiseltest repo and run the unit tests for iverilog / Verilator as the simulator backend
        - Inspect the files that chiseltest generates to understand how we communicate between Scala (JVM) and C++
    - Run alu.v on verilator (see Slack for the files)
    - Set up a strawman repo where you use JNA to invoke a dummy C library from the JVM
        - In isolation, set up a strawman of JNI and JNA for a simple C API that does basically nothing to measure FFI overhead
        - https://github.com/java-native-access/jna/blob/master/www/GettingStarted.md
        - Start with JNA, work through their docs and use a bare Scala project with sbt
    - Next: do the same with JNI and measure the performance difference using JMH
    - Next next: do the same with the new Java 19 FFM API and benchmark that version as well (https://openjdk.org/jeps/424)
    - Read over the VPI docs (https://en.wikipedia.org/wiki/Verilog_Procedural_Interface)
        - set up a simple example with iverilog (https://iverilog.fandom.com/wiki/Using_VPI)
    - Skim some chiseltest code
        - https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/simulator/jna/JNAUtils.scala
        - https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/simulator/jna/JNASimulatorContext.scala
        - https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/simulator/VerilatorSimulator.scala
        - https://github.com/ucb-bar/chiseltest/blob/main/src/main/scala/chiseltest/simulator/IcarusSimulator.scala
