[`AbstractProcessor`](https://hazelcast-l337.ci.cloudbees.com/view/Jet/job/Jet-javadoc/javadoc/com/hazelcast/jet/core/AbstractProcessor.html)
is a convenience class designed to deal with most of the boilerplate in
implementing the full `Processor` API.

## Receiving items

On the reception side the first line of convenience are the
`tryProcessN()` methods. While in the inbox the watermark and data items
are interleaved, these methods take care of the boilerplate needed to
filter out the watermarks. Additionally, they get one item at a time, 
eliminating the need to write a suspendable loop over the input items.

There is a separate method specialized for each edge from 0 to 4
(`tryProcess0`..`tryProcess4`) and a catch-all method
`tryProcess(ordinal, item)`. If the processor doesn't need to
distinguish between the inbound edges, the latter method is a good
match; otherwise, it is simpler to implement one or more of the
ordinal-specific methods. The catch-all method is also the only way to
access inbound edges beyond ordinal 4, but such cases are very rare in
practice.

Paralleling the above there are `tryProcessWm(ordinal, wm)` and 
`tryProcessWmN(wm)` methods that get just the watermark items.

## Emitting items

`AbstractProcessor` has a private reference to its outbox and lets you 
access all its functionality indirectly. The `tryEmit()` methods offer your items to the outbox. If you get a `false` return value, you must stop emitting items and return from the current callback method of the processor. For example, if you called `tryEmit()` from `tryProcess0()`,
you should return `false` so Jet will call `tryProcess0()` again later, when there's more room in the outbox. Similar to these methods there are `tryEmitToSnapshot()` and `emitFromTraverserToSnapshot()`, to be used from the `saveToSnapshot()` callback.

Implementing a processor to respect the above rule is quite tricky and error-prone. Things get especially tricky when there are several items to emit, such as:

- when a single input item maps to many output items
- when the processor performs a group-by-key operation and emits each
group as a separate item

You can avoid most of the complexity if you wrap all your output in a `Traverser`. Then you can simply say `return
emitFromTraverser(myTraverser)`. It will:

1. try to emit as many items as possible
2. return `false` if the outbox refuses an item
3. hold on to the refused item and continue from it when it's called 
   again with the same traverser.

There is one more layer of convenience relying on `emitFromTraverser()`:
the nested class `FlatMapper`, which makes it easy to implement a
flatmapping kind of transformation. It automatically handles the concern
of creating a new traverser when the previous one is exhausted and
reusing the previous one until exhausted.

## Traverser

`Traverser` is a very simple functional interface whose shape matches
that of a `Supplier`, but with a contract specialized for the traversal
over a sequence of non-null items: each call to its `next()` method
returns another item of the sequence until exhausted, then keeps
returning `null`. A traverser may also represent an infinite,
non-blocking stream of items: it may return `null` when no item is
currently available, then later return more items.

The point of this type is the ability to implement traversal over any 
kind of dataset or lazy sequence with minimum hassle: often just by 
providing a one-liner lambda expression. This makes it very easy to 
integrate with Jet's convenience APIs for cooperative processors.

`Traverser` also supports some `default` methods that facilitate
building a simple transformation layer over the underlying sequence:
`map`, `filter`, `flatMap`, etc.

The following example shows how you can implement a simple flatmapping
processor:

```java
public class ItemAndSuccessorP extends AbstractProcessor {
    private final FlatMapper<Integer, Integer> flatMapper =
        flatMapper(i -> traverseIterable(asList(i, i + 1)));

    @Override
    protected boolean tryProcess(int ordinal, Object item) {
        return flatMapper.tryProcess((int) item);
    }
}
```

For each received `Integer` this processor emits the number and its 
successor. If the outbox refuses an item, `flatMapper.tryProcess()` 
returns `false` and stays ready to resume the next time it is invoked. 
The fact that it returned `false` signals Jet to invoke 
`ItemAndSuccessorP.tryProcess()` again with the same arguments.
