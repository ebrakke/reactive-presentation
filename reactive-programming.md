## What is reactive programming?

As Andre Staltz (creator of cycle.js) puts it: <!-- .element: class="fragment" -->

Reactive programming is programming with asynchronous data streams. <!-- .element: class="fragment" -->

Reactive libraries allow you to create data streams for anything <!-- .element: class="fragment" -->

You can listen to these streams, and "react" when data arrives on that stream <!-- .element: class="fragment" -->

Note:
* These data streams already exist in the DOM, like clickEvents and navigation events

--

## Utilities for streams

Reactive libraries like RxJS come with functions to manipulate streams <!-- .element: class="fragment" -->

You can merge two streams together or filter a stream to only emit when a condition is met <!-- .element: class="fragment" -->

--

## Example stream

```
Web socket stream:
---1----2--1--------4-4--|-->

A stream that emits numbers
completes after the last 4 is emitted
```

What can we do with this stream?

--

We can use utility functions to transform this stream into a different stream

```
---1----2--1--------4-4--|-->

VVV filter (x => x > 1)   VVV

--------2-----------4-4--|-->
```

Now if we had a use case in our app where want to show an alert if this web socket emits a value above 1, we can just subscribe to the new stream

--

## Detecting a double click

```
----c--c------cc----cccc----c----c--->

VVVV        bufferTime(250)       VVVV

----c--c----(cc)----(cccc)--c----c--->

VVV filter(clicks => clicks.length > 2) VVV
------------(cc)----(cccc)----------->
```

So if want to respond to a user double clicking an element, we can use the resulting stream

Note:
* Theres no limit to what can be a stream
* Buffertime will gather events in the window and then emit

--

## Streams are lazy

--

Define complex data calls and manipulations, but never evaluate until someone subscribes to the data

```ts
const startHeavyCalc$ = fromEvent('click', '.button')
const result$ = startHeavyCalc$.pipe(
    switchMap(() => api.someExpensiveCall())
)
const derivedFromResult$ = result$.pipe(
    map(data => data * 2)
)
```

None of this code will run until `.subscribe` is called on any of the observables <!-- .element: class="fragment -->

--

## Streams as state

Streams have a concept of the "last emitted value", and streams can also be "shared"

```ts
const result$ = myApi.fetchData$.pipe(shareReplay(1))

const sub1 = result$.subscribe(v => console.log(v))
const sub2 = result$.subscribe(v => console.log(v))
```

sub1 and sub2 will receive the same data, and only one network request will occur on the initial subscription <!-- .element: class="fragment" -->

Note:
* Shared means that new subscribers will share the same data stream, rather than a new data stream
* Lazy nature means that we won't make unnecessary calls to load data