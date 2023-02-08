# Chiseltest FFI Performance


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
