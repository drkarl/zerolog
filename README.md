<img src="https://raw.githubusercontent.com/wiki/obsidiandynamics/zerolog/images/zerolog-logo.png" width="90px" alt="logo"/> ZeroLog
===
[![Download](https://api.bintray.com/packages/obsidiandynamics/zerolog/zerolog-core/images/download.svg) ](https://bintray.com/obsidiandynamics/zerolog/zerolog-core/_latestVersion)
[![Build](https://travis-ci.org/obsidiandynamics/zerolog.svg?branch=master) ](https://travis-ci.org/obsidiandynamics/zerolog#)
[![codecov](https://codecov.io/gh/obsidiandynamics/zerolog/branch/master/graph/badge.svg)](https://codecov.io/gh/obsidiandynamics/zerolog)
===
Low-overhead logging façade for performance-sensitive applications.

# What is ZeroLog?
ZeroLog (abbreviated to _Zlg_) is a logging façade with two fundamental design objectives:

1. **Ultra-low overhead for suppressed logging.** In other words, the cost of calling a log method when logging for that level has been disabled is negligible.
2. **Uncompromised code coverage.** Suppression of logging should not impact statement and branch coverage metrics. A log entry is a statement like any other.

Collectively, these objectives make Zlg suitable for use in ultra-high performance, low-latency applications and in high-assurance environments.

# How fast is it?
A JMH benchmark conducted on an [i7-4770 Haswell](https://ark.intel.com/products/75122/Intel-Core-i7-4770-Processor-8M-Cache-up-to-3_90-GHz) CPU with logging suppressed compares the per-invocation penalties for Zlg with some of the major loggers. Four primitives are passed to each logger for formatting, which is a fair representation of a typical log entry.

Logger implementation       |Avg. time (ns)
:---------------------------|-------------:
JUL (java.util.logging)     |12.067        
JUL w/ lambda               |0.141         
Log4j 1.2.17                |9.577         
SLF4J 1.7.25 w/ Log4j 1.2.17|14.298        
TinyLog 1.3.4               |11.770        
Zlg                         |0.060  

To replicate this benchmark on your machine, run `./gradlew launch -Dlauncher.class=AllBenchmarks`.

# Getting Started
## Dependencies
Gradle builds are hosted on JCenter. Just add the following snippet to your build file. Replace the version placeholder `x.y.z` in the snippet with the version shown on the Download badge at the top of this README.

```groovy
compile "com.obsidiandynamics.zerolog:zerolog-core:x.y.z"
compile "com.obsidiandynamics.zerolog:<binding>:x.y.z"
```

You need the `zerolog-core` module and, typically, a binding. The sole currently supported Zlg binding is `zerolog-slf4j17`, which should work with any logger that features an SLF4J 1.7.x binding. (This covers all major loggers.) For example, to use Zlg with Log4j 1.2.17, add the following to your `build.gradle` (replacing `x.y.z` as appropriate).

```groovy
compile "com.obsidiandynamics.zerolog:zerolog-core:x.y.z"
compile "com.obsidiandynamics.zerolog:zerolog-slf4j17:x.y.z"
runtime "org.slf4j:slf4j-api:1.7.25"
runtime "org.slf4j:slf4j-log4j12:1.7.25"
runtime "log4j:log4j:1.2.17"
```

**Note:** `zerolog-slf4j17` doesn't declare a specific `slf4j-api` version Maven dependency, allowing you to nominate _any_ binary-compatible SLF4J API in your project. The upshot is that you have to explicitly include the `slf4j-api` version in your build file. This can be relegated to the `runtime` configuration (unless you need to use SLF4J directly in your application, alongside Zlg).

## Logging
Getting a logger instance isn't too different from SLF4J. Typically, a named logger instance is first obtained from a factory and subsequently assigned to either an instance or a static field, as shown below.

```java
public final class SysOutLoggingSample {
  private static final Zlg zlg = Zlg.forClass(SysOutLoggingSample.class).get();
  
  public static void open(String address, int port, double timeoutSeconds) {
    zlg.i("Connecting to %s:%d [timeout: %.1f sec]").arg(address).arg(port).arg(timeoutSeconds).log();
    try {
      openSocket(address, port, timeoutSeconds);
    } catch (IOException e) {
      zlg.w("Error connecting to %s:%d").arg(address).arg(port).tag("I/O").threw(e).log();
    }
  }
}
```

Some important things to note:

* A logger is a `Zlg` instance, created for a specific class (`forClass()`) or an arbitrary name (`forName()`). By convention, we name the field `zlg`.
* Logging is invoked via a fluent call chain, starting with the log level (abbreviated to the first letter) specifying a mandatory format string, any optional arguments (primitives or object types), an optional tag, and an optional exception.
* Each chain _must_ end with a `log()` for the log entry to be printed.
* The format string is printf-style, unlike most other loggers that use the `{}` (stash) notation.

# Tags
Zlg adds the concept of a _tag_ — an optional string value that can be used to decorate a log entry. A tag is equivalent to a marker in SLF4J, adding another dimension for slicing and dicing your log output.

# Log levels
Zlg log levels are reasonably well-aligned with SLF4J (and most other loggers). Zlg introduces a new log level — `LogLevel.CONF` — logically situated between `DEBUG` and `INFO`. Loosely borrowed from JUL (`java.util.logging`), `CONF` is intended for logging initialisation and configuration parameters, useful when offering a variety of configuration options to the user.

**Note:** `CONF` is canonically mapped to `INFO` in those loggers that don't support `CONF` directly.

The built-in log levels are, from lowest to highest: `TRACE`, `DEBUG`, `CONF`, `INFO`, `WARN`, `ERROR` and `OFF`. 

**Note:** `OFF` is not a legal log level, insofar as it cannot be used to output a log entry from the application code; it's use purely for configuration — being the highest of all levels, **`OFF` is used to disable logging altogether**.

# Configuration
## Bindings
Being a façade, Zlg delegates all log calls to an actual logger — an implementation of `LogService`. By default, Zlg comes pre-packaged with a very basic 'failsafe' `SysOutLogService` that prints entries to `System.out` in a fixed format. Example below.

```
21:23:16.814 INF [main]: Connecting to github.com:80 [timeout: 30.0 sec]
21:23:16.818 WRN [main] [I/O]: Error connecting to github.com:80
```

Zlg detects installed bindings using Java's [SPI](https://docs.oracle.com/javase/tutorial/ext/basics/spi.html) plugin mechanism. By simply including a binding on the classpath, Zlg will switch over to the new binding by default.

## Logger configuration
Like SLF4J, Zlg is largely hands-off when it comes to logger configuration management, leaving the configuration specifics to the bound logger implementation. Configuration Log4j 1.2, for example, would be done through `log4j.properties` — Zlg remains completely agnostic of this.

## Baseline configuration with `zlg.properties`
Zlg supports a _baseline_ configuration, by reading an optional `zlg.properties` file from the classpath. A baseline configuration comprises a list of optional properties. For example, it specifies the base log level (below which all logging is disabled) and can override the default binding. The following is an example of `zlg.properties`.

```
zlg.base.level=CONF
zlg.log.service=com.obsidiandynamics.zerolog.SysOutLogService
```

The `zlg.base.level` property specifies the minimum enabled log level (irrespective of what may be allowed by the bound logger). The default value is `CONF`, meaning that **unless the baseline is altered, `TRACE` and `DEBUG` entries will be ignored**. The choice of `CONF` as the default level is closest to a typical production configuration.

While `zlg.properties` is optional, it is strongly recommended that `zlg.properties` is present during development with a single `zlg.base.level` entry. Practically, this could be a file in `src/main/resources` or `src/test/resources` with an appropriate VCS ignore rule, so that changes to `zlg.properties` aren't committed back to your VCS. This allows developers to enable/disable debugging for local development without stuffing around with the logger configuration or accidentally committing back the change. Excluding it from VCS means that it will be absent from your CI build or a clean checkout build; this is normally quite acceptable, as the default configuration is geared towards production use.

## Changing the location of `zlg.properties`
The default location of `zlg.properties` can be overridden by setting the `zlg.default.config.uri` system property. The default URI is `cp://zlg.properties`, `cp://` denoting 'classpath'. Alternatively, a file system location can be specified with the `file://` scheme.

When overriding the default file location, ideally the `zlg.default.config.uri` property is passed in as a `-D...` JVM argument, ensuring that the logging subsystem is bootstrapped correctly before initial use.

## In-line configuration
In addition to `zlg.properties`, Zlg supports in-line configuration at the point when the logger is obtained:

```java
final Zlg zlg = Zlg
    .forClass(MyAppClass.class)
    .withConfigService(new LogConfig().withBaseLevel(LogLevel.TRACE))
    .get();
```

In-line configuration assumes priority, overriding any system-default values or values provided by `zlg.properties`. So, if you wanted to force a specific class to log to the console while using an SLF4J binding for all other classes, you could just do this:

```java
final Zlg zlg = Zlg
    .forClass(MyAppClass.class)
    .withConfigService(new LogConfig()
                       .withBaseLevel(LogLevel.TRACE)
                       .withLogService(new SysOutLogService()))
    .get();
```


# FAQ
## Aren't there enough loggers already?
Zlg isn't a logger, it is a _logging façade_, acting as an interface between your application code and the logger implementation. It is up the underlying logger to format and persist (or forward) the logs.

Zlg comes with a 'failsafe' logger — `SysOutLogService`; however, this is only a stop-gap measure until you install an appropriate log binding.

## Okay, aren't there enough façades already?
There is one _de facto_ façade — SLF4J. (Ignoring Apache Commons Logging, as it's effectively obsolete.) SLF4J is an excellent all-round library and one that serves an overwhelming majority of use cases. However, as the benchmarks reveal, the penalty of invoking SLF4J in suppressed mode is in the order of 10 ns, which can be substantial for highly performant code and tight loops. When working with sub-microsecond applications, suppressed logging accounts for more than 1% of the cost, which may be unacceptably high.

Zlg was designed with one goal — materially reduce the cost of suppressed logging, but without sacrificing code coverage or eroding maintainability.

SLF4J supports guards, so overhead can be driven to a minimum with something like this:

```java
if (logger.isTraceEnabled()) logger.trace("float: {}, double: {}, int: {}, long: {}", f, d, i, l);
```

This presents a few problems, notably it —

* Introduces a branching instruction, which affects code coverage.
* Leads to code duplication, as the log level must be specified twice.
* Is susceptible to copy-paste errors; we've all accidentally done this before: `if (logger.isDebugEnabled()) logger.warn("Something just happened")`.
* Imposes a double-checked penalty if logging is enabled.

Conversely, a Zlg statement doesn't repeat the level, doesn't introduce a branch statement and doesn't double-check if logging is enabled.

## Why is it so fast?
A better question to ask is why typical loggers and logging façades are slow. A typical SLF4J statement (other loggers are mostly in the same boat) looks like this:

```java
logger.trace("float: {}, double: {}, int: {}, long: {}", f, d, i, l);
```

There are several problems with this approach, each a substantial performance drain.

* **Use of varargs to pass parameters.** Typically beyond two or three formatting arguments, loggers will offer a varargs-based API to accommodate arbitrary number of arguments. Varargs are just syntactic sugar; the performance impact is that of array allocation. Furthermore, any escape analysis done by the optimiser will conclude that the array could be used beyond the scope of the method call and thus stack allocation will not be possible — a full heap allocation will ensue.

* **Boxing of primitive types.** Formatting arguments are either `Object` types or vararg arrays of `Object`. Passing primitives to the API will result in autoboxing. The autobox cache is typically quite minuscule; only a relatively small handful of primitives are interned. Even the interned primitives still require branching, offset arithmetic and an array lookup to a resolve. (See `Integer.valueOf(int)` for how this is implemented in the JDK.)

* **Garbage collection.** Both varargs and autoboxing ultimately allocate lots of objects irrespective of whether logging is enabled or suppressed, especially when logging is done from tight loops. This reduces the application throughput and increases latency by introducing periodic pauses.

Zlg solves the above problems by accumulating primitives one-at-a-time into an instance of `Zlg.LogChain`. The `LogChain` API has an `arg()` method for each of Java's eight primitive types, as well as `Object`. (There could be any number of inter-mixed primitives and object types, in any combination, and designing a single interface with all possible combinations of relevant data types is an exponential function of the arity of the argument domain. With four arguments, there will be 6,561 distinct methods. This would require some form of code generation to achieve, and would certain kill any IDE auto-complete feature. With seven arguments, this number is around four million.) By using the fluent chaining patter, and dealing with one argument at a time, Zlg circumvents the arity problem with a tiny interface. When logging is suppressed, primitives are never boxed and arrays are never allocated — thus Zlg has a zero object allocation rate and zero impact on the GC.

Interface-driven design also enables Zlg to substitute a log chain with a `NopLogChain` as soon as it determines that logging has been disabled for the requested level. Because the `NopLogChain` is a no-op, JIT is able to aggressively inline most of the chained calls. In our benchmarks we see no material difference between short and long chains; evidently JIT is doing its job.

## Where should I use Zlg?

## Should I replace all uses of SLF4J with Zlg?

## Why isn't a Tag called a Marker?

## Where are the bindings for other loggers?

## When delegating to SLF4J, is class/method/line location information preserved?

## Why use printf-style formatting?




## I don't care about coverage, can I have a true _zero_-footprint logger?
If sub-nanosecond penalties for suppressed logs are still too high and you require true zero, your only option is to strip out the logging instructions altogether. Fortunately, you don't need to do this during compilation; JIT DCE (dead code elimination) can intelligently do this for you on the fly. There are a couple of patterns that achieve zero footprint in different ways.

**1. Branching on a static constant** — will lead to DCE for one of the branches. Example:

```java
private static final Zlg zlg = Zlg.forClass(MethodHandles.lookup().lookupClass()).get();

private static final boolean TRACE_ENABLED = false;

public static void withStaticConstant(String address, int port, double timeoutSeconds) {
  if (TRACE_ENABLED) zlg.t("Connecting to %s:%d [timeout: %.1f sec]").arg(address).arg(port).arg(timeoutSeconds).log();
}
```

**2. Assertions** — when running with `-ea` logging instructions will be evaluated; otherwise they will be DCE'ed. Example:

```java
private static final Zlg zlg = Zlg.forClass(MethodHandles.lookup().lookupClass()).get();

public static void withAssert(String address, int port, double timeoutSeconds) {
  assert zlg.t("Connecting to %s:%d [timeout: %.1f sec]").arg(address).arg(port).arg(timeoutSeconds).logb();
}
```

**Note:** Rather than using `log()`, the assertion example uses `logb()`, which works identically to `log()` but returns a constant `true`. If assertions are enabled with the `-ea` JVM argument, the log instruction will be evaluated and will never fail the assertion. Otherwise, the entire fluent chain will be dropped due by DCE.

The choice of using option one or two depends on whether you are targeting zero overhead for both production and testing scenarios or only for production. In case of the latter, the `-ea` flag naturally solves the problem, without forcing you to change your class before building. In either case, you will sacrifice code coverage, as both techniques introduce a parasitic branching instruction behind the scenes; only one path is traversed during the test.

## Can Zlg be mocked?
Zlg's design is heavily interface-driven, for two main reasons. The first is to simplify mocking and testing, which in itself allows us to maintain Zlg with 100% instruction and branch coverage. The following is an example of mocking various parts of the log chain with Mockito 2.18:

```java
final Zlg zlg = mock(Zlg.class, Answers.CALLS_REAL_METHODS);
final LogChain logChain = mock(LogChain.class, Answers.CALLS_REAL_METHODS);
when(logChain.format(any())).thenReturn(NopLogChain.getInstance());
when(zlg.level(anyInt())).thenReturn(logChain);

zlg.t("the value of Pi is %.3f").arg(Math.PI).log();
```

**Note:** If you simply want a no-op logger to suppress any potential output, and don't care about mocking the fluent call chain, you can instantiate the logger with `LogLevel.OFF`:

```java
Zlg.forName("no-op").withConfigService(new LogConfig().withBaseLevel(LogLevel.OFF)).get();
```

The second reason for reliance on interface is not quite so banal, and more driven by performance. It enables Zlg to substitute a log chain with a `NopLogChain` as soon as it determines that the logging has been disabled for the requested level, thereby inlining most of the fluent calls.