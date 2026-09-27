---
title: "(Notes) Go GC algorithm"
date: "2023-12-28"
weight: 10
---

Go is a language with automatic GC (garbage collection). The GC process uses a tri-color mark-and-sweep algorithm.

### Tri-color algorithm

**Two stages**

1. Mark phase: mark objects that are reachable. It's a directed graph problem. You start from a set of root objects (e.g. global variables, pointers from the stack to the heap) and process all objects referenced directly or indirectly.
2. Sweep phase: reclaim objects that are not reachable. This doesn't happen synchronously but amortizes over new memory allocations.

**Three colors**

- White: Not reachable.
- Grey: Reachable, immediate descendants not yet processed
- Black: Reachable, immediate descendants processed

**Write barriers**

- GC runs concurrently with the application code, only pausing the application code when necessary. So it's not a static graph, but a dynamic one. If a black object is modified to point to a white object, the white object is marked grey.
- One interesting property of the graph is that, once an object becomes unreachable, there will never be new references to it. Why? Because it's unreachable!
- Note that you can't mark it as black because you haven't processed its descendants yet. This is why you need three colors instead of two.

**Delete barriers**

- Mark an object as grey if a reference is removed from it. This is not always necessary but a precaution. This is used in some versions of Go but I'm not sure which one.

**Mark phase breakdown (for Go in 2015)**

1. Mark setup: STW (Stop the world). Turn on the write barrier. Scan the stacks to get pointers to the heap.
2. Mark: Application code is resumed. GC goroutine does the marking concurrently.
3. Mark termination: STW. Turn off the write barrier.

### Tuning Go GC

https://tip.golang.org/doc/gc-guide

`GOGC=100` means start GC when additional allocation is 100% of the live heap size.

`GOMEMLIMIT` specifies a soft cap on memory allocation.

### References

https://community.apinto.com/d/34057-golang-gc

https://go.dev/talks/2015/go-gc.pdf

https://www.ardanlabs.com/blog/2019/05/garbage-collection-in-go-part2-gctraces.html

https://community.apinto.com/d/34057-golang-gc
