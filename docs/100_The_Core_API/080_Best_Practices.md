## Inspecting Processor Input and Output

The structure of the DAG model is a very poor match for Java's type
system, which results in the lack of compile-time type safety between
connected vertices. Developing a type-correct DAG therefore usually
requires some trial and error. To facilitate this process, but also to
allow many more kinds of diagnostics and debugging, Jet's library offers
ways to capture the input/output of a vertex and inspect it.

### Peeking with Processor Wrappers

The first approach is to decorate a vertex declaration with a layer that
will log all the data traffic going through it. This support is present
in the
[`DiagnosticProcessors`](https://hazelcast-l337.ci.cloudbees.com/view/Jet/job/Jet-javadoc/javadoc/com/hazelcast/jet/core/processor/DiagnosticProcessors.html)
factory class, which contains the following methods:

* `peekInput()`: logs items received at any edge ordinal.

* `peekOutput()`: logs items emitted to any ordinal. An item emitted to 
  several ordinals is logged just once.

These methods take two optional parameters:

* `toStringF` returns the string representation of an item. The default
  is to use `Object.toString()`.
* `shouldLogF` is a filtering function so you can focus your log output
  only on some specific items. The default is to log all items.

#### Example Usage

Suppose we have declared the second-stage vertex in a two-stage
aggregation setup:

```java
Vertex combine = dag.newVertex("combine", 
    combineByKey(counting()));
```

We'd like to see what exactly we're getting from the first stage, so
we'll wrap the processor supplier with `peekInput()`:

```java
Vertex combine = dag.newVertex("combine", 
    peekInput(combineByKey(counting())));
```

Keep in mind that logging happens on the machine running hosting the
processor, so this technique is primarily targetted to Jet jobs the
developer runs locally in his development environment.

### Attaching a Sink Vertex

Since most vertices are implemented to emit the same data stream to all
attached edges, it is usually possible to attach a diagnostic sink to
any vertex. For example, Jet's standard
[`writeFileP()`](https://hazelcast-l337.ci.cloudbees.com/view/Jet/job/Jet-javadoc/javadoc/com/hazelcast/jet/core/processor/SinkProcessors.html#writeFileP-java.lang.String-)
sink vertex can be very useful here.

#### Example Usage

In the example from the Word Count tutorial we can add the following
declarations:

```java
Vertex diagnose = dag.newVertex("diagnose",
        Sinks.writeFile("tokenize-output"))
        .localParallelism(1);
dag.edge(from(tokenize, 1).to(diagnose));
```

This will create the directory `tokenize-output` which will contain one
file per processor instance running on the machine. When running in a
cluster, you can inspect on each member the input seen on that member.
By specifying the `allToOne()` routing policy you can also have the
output of all the processors on all the members saved on a single member
(although the choice of exactly which member will be arbitrary).

## How to Unit-Test a Processor

We provide some utility classes to simplify writing unit tests for your custom processors. You can find them in the
[`com.hazelcast.jet.core.test`](https://hazelcast-l337.ci.cloudbees.com/view/Jet/job/Jet-javadoc/javadoc/com/hazelcast/jet/core/test/package-summary.html)
package. Using these utility classes you can unit test your processor by
passing it some input items and asserting the expected output.

Start by calling
[`TestSupport.verifyProcessor()`](https://hazelcast-l337.ci.cloudbees.com/view/Jet/job/Jet-javadoc/javadoc/com/hazelcast/jet/core/test/TestSupport.html#verifyProcessor-com.hazelcast.jet.core.ProcessorSupplier-)
by passing it a processor supplier or a processor instance.

The test process does the following:

* initialize the processor by calling `Processor.init()`
* do a snapshot+restore (optional, see below)
* call `Processor.process(0, inbox)`. The inbox always contains one
  item from the `input` parameter
* every time the inbox gets empty, do a snapshot+restore
* call `Processor.complete()` until it returns `true` (optional)
* do a final snapshot+restore after `complete()` is done

The optional snapshot+restore test procedure:

* call `saveToSnapshot()`
* create a new processor instance and use it instead of the existing one
* restore the snapshot using `restoreFromSnapshot()`
* call `finishSnapshotRestore()`

The test optionally asserts that the processor made progress on each call to any processing method. To be judged as having made progress, the callback method must do at least one of these:

* take something from the inbox
* put something to the outbox
* return `true` (applies only to `boolean`-returning methods)

#### Cooperative Processors

The test will provide a 1-capacity outbox to cooperative processors. The
outbox will already be full on every other call to `process()`. This
tests the edge case: `process()` may be called even when the outbox is
full, giving the processor a chance to process the inbox without
emitting anything.

The test will also assert that the processor doesn't spend more time in
any callback than the limit specified in `cooperativeTimeout(long)`.

#### Cases Not Covered

This class does not cover these cases:

* testing of processors which distinguish input or output edges by
  ordinal
* checking that the state of a stateful processor is empty at the end
  (you can do that yourself afterwards with the last instance returned
  from your supplier)
* it never calls `Processor.tryProcess()`

#### Example Usage

This will test one of the jet-provided processors:

```java
TestSupport.verifyProcessor(Processors.map((String s) -> s.toUpperCase()))
           .disableCompleteCall()             // enabled by default
           .disableLogging()                  // enabled by default
           .disableProgressAssertion()        // enabled by default
           .disableSnapshots()                // enabled by default
           .cooperativeTimeout(<timeoutInMs>) // default is 1000
           .outputChecker(<function>)         // default is `Objects::equal`
           .input(asList("foo", "bar"))       // default is `emptyList()`
           .expectOutput(asList("FOO", "BAR"));
```

### Other Utility Classes

`com.hazelcast.jet.test` contains these classes that you can use as
implementations of Jet interfaces in tests:

* `TestInbox`
* `TestOutbox`
* `TestProcessorContext`
* `TestProcessorSupplierContext`
* `TestProcessorMetaSupplierContext`

The class `JetAssert` contains a few of the `assertX()` methods normally
found in JUnit's `Assert` class. We had to reimplement them to avoid a
dependency on JUnit from our production code.
