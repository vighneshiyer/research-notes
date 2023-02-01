# Chiseltest FFI Performance

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
