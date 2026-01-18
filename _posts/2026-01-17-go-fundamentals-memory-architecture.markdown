---
date: 2026-01-17 12:00:01
title: "Go Fundamentals: Memory Architecture"
description: Understanding stack vs heap, escape analysis, garbage collection, and memory performance in Go
image: images/posts/go-fundamentals/go-fundamentals-memory-architecture.png
categories: [go]
tags: [go, fundamentals, memory, stack, heap, garbage-collection, performance]
series: go-fundamentals
---

This post covers Go memory architecture, including the call stack, heap allocation, escape analysis, garbage collection, and performance considerations for pointers.

<div class="series-nav">
  <div class="series-nav__title">Go Fundamentals Series</div>
  <div class="series-nav__subtitle">Article 2 of 4</div>
  <div class="series-nav__list">
    <a href="/go-fundamentals-variable-behavior" class="series-nav__item">1. Variable Behavior</a>
    <span class="series-nav__item--current">2. Memory Architecture ← You are here</span>
    <a href="/go-fundamentals-concurrency-model" class="series-nav__item">3. Concurrency Model</a>
    <a href="/go-fundamentals-error-handling" class="series-nav__item">4. Error Handling</a>
  </div>
</div>

## Call Stack

In programming languages in general, and in Go specifically, runtime threads use stacks to execute code.

### Thread Stack Example

When running code and placing a breakpoint, we can observe the stack trace showing called functions, and each function has its own frame of local variables.

![Thread Stack Debug View](/images/posts/go-fundamentals/thread-stack-debug.png)

### Thread Stack Anatomy

The stack grows as functions are called, with each function adding its own frame containing arguments, local variables, and return values:

![Thread Stack Anatomy](/images/posts/go-fundamentals/thread-stack-anatomy.png)

### Returning Values

How will the stack look for a function returning a value?

```go
func main() {
    y := func1()
}

func func1() int {
    y := 2
    return y * 2
}
```

After `func1` returns, `main` receives the value `y = 4`.

![Stack Returning Value - Before](/images/posts/go-fundamentals/stack-returning-value-before.png)

![Stack Returning Value - After](/images/posts/go-fundamentals/stack-returning-value-after.png)

### Passing a Pointer to a Local Variable

How will the stack look for a function passing a pointer to a local variable?

```go
func main() {
    x := 0
    func1(&x)
}

func func1(res *int) {
    y := 2
    *res = y * 2
}
```

The pointer allows `func1` to modify `main`'s local variable directly.

![Stack Pointer Up](/images/posts/go-fundamentals/stack-pointer-up.png)

### Returning a Pointer to a Local Variable

How will the stack look for a function returning a pointer to a local variable?

```go
func main() {
    x := func1()
}

func func1() *int {
    y := 2
    res := y * 2
    return &res
}
```

![Stack Pointer Down](/images/posts/go-fundamentals/stack-pointer-down.png)

And then what happens when we call another function?

```go
func main() {
    x := func1()
    func2()
    // x now points to... what?
}
```

![Stack Pointer Dangling](/images/posts/go-fundamentals/stack-pointer-dangling.png)

This is where the heap comes in.

## The Heap

The heap stores all values that can't live on a thread stack:

- Global variables
- Returned pointers
- Values shared between threads
- ...

![Heap Address Space](/images/posts/go-fundamentals/process-address-space.png)

## Allocation and Deallocation

**Stack operations are very cheap** because the stack is linear. Holding an int pointing to the top of the stack is enough. Then we increment and decrement this value to allocate and deallocate.

**Heap operations are expensive**. Allocation is an OS operation which requires looking up free space and marking it as used. Deallocation is also not cheap.

But when can we remove objects from the heap? The stack is easy. The heap requires explicit handling:
- In C/C++ we explicitly free up heap objects
- In Rust the compiler does it for us
- In Go we have **garbage collection**

This typically has additional CPU overhead of 10s of %.

## Heap vs Stack

Whether a Go value is stored on the stack or escapes to the heap is a function of the Go compiler's **escape analysis** algorithm. The algorithm itself changes between Go releases to support new optimizations.

Example use cases:

```go
var v = 32
func sliceAllocateWithMaxElementSumSize32Var() {
    var x = make([]byte, v)    // escapes to heap
    var y = make([]byte, v+1)  // escapes to heap
    {...}
}

const q = 32
func sliceAllocateWithMaxElementSumSize32() {
    var x = make([]byte, q)    // on stack
    var y = make([]byte, q+1)  // on stack
    {...}
}

const M = 10 * 1024 * 1024
func sliceAllocateWithMaxElementSize10m() {
    var a1 [M]byte      // on stack
    var a2 [M + 1]byte  // escapes to heap
    {...}
}

const N = 65536
func sliceAllocateWithMaxElementSize64k() {
    var a1 = new([N]byte)      // on stack
    var a2 = new([N + 1]byte)  // escapes to heap
    {...}
}
```

## Escape Analysis Tools

Go offers tooling for analysis of heap allocation and variables escaping to the heap.

### 1. Escape Analysis Tool

Build with `go build -gcflags '-m -l'` to get a list of escaping done by the compiler.

![Escape Analysis Tool Result](/images/posts/go-fundamentals/escape-analysis-tool-result.png)

### 2. Benchmark Testing

- File must end with `_test`
- Function must start with `Benchmark` and accept `(b *testing.B)`
- Iterate over benchmark iterations and call the function you'd like to benchmark

```go
func BenchmarkMain1(b *testing.B) {
    for i := 0; i < b.N; i++ {
        main1()
    }
}
```

Run `go test -bench . -benchmem` from the terminal:

```
$ go test -bench . -benchmem
goos: darwin
goarch: amd64
pkg: infra-guild
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkMain1-12    965051588    1.229 ns/op    0 B/op    0 allocs/op
PASS
ok    infra-guild    1.925s
```

## Garbage Collector

How does the Garbage Collector work?

Go has a **mark-sweep GC**. It operates in two phases:

1. **Mark phase**: Scans the entire heap looking for live objects and marking them
2. **Sweep phase**: Removes all unmarked objects

The scanning process begins at all nodes considered **GC roots**, and recursively follows all pointers down to the leaves. All reachable nodes are considered live.

Examples of GC roots:
- Stack variables
- Global variables

Further reading [here](https://tip.golang.org/doc/gc-guide#Tracing_Garbage_Collection).

## Memory Leak

A memory leak in a garbage collected language is when an unused piece of memory is reachable from a GC root, causing it to be considered live forever. This causes the memory to keep growing indefinitely.

### Memory Leak Example

We were using an in-memory cache library called `gcache`:
- The gcache library supports setting TTL for values
- We used the cache by writing values with TTL to it
- In production we saw OOM crashes, and memory graphs kept going upwards
- We ran profiling to get memory dumps and saw that the cache was only ever growing
- We examined the source code of gcache to realize it implements a **lazy TTL mechanism** - expired entries are removed only when we try to read them

## Performance Testing

Take a look at `pass_by_pointer` and `pass_by_value` and compare the two. What is the difference?

```go
func passByValue(s LargeStruct) int {
    return s.field1 + s.field2
}

func passByPointer(s *LargeStruct) int {
    return s.field1 + s.field2
}
```

Run benchmark on both of them to see the performance difference. The results may surprise you - it depends on the size of the struct and how it's used.

To understand why, run profiling on both of them.

### How to Read Profiling Results

![Profiling Results](/images/posts/go-fundamentals/profiling-results.png)

The flame graph shows you where time is being spent in your code, helping you identify performance bottlenecks.

## When To Use Pointers

When to use pointers (including pointer receivers, i.e. `this` variable in methods):

1. **When you need mutability** - the function needs to modify the original value
2. **When another method is already using a pointer receiver** - a common standard: if one method needs a pointer receiver, all methods should use pointer receivers
3. **When you have to allow nil values**
4. **In terms of performance** - always use benchmarks and profiling to determine

For example, when working with large structs that have long lifecycles or have large amounts of function call passing, pointers will probably perform better.

## Further Reading

- [When to use receiver pointers in Go](https://golang.org/doc/faq#methods_on_values_or_pointers)
- [Go Garbage Collector](https://tip.golang.org/doc/gc-guide)
- [Understanding Allocations in Go](https://medium.com/eureka-engineering/understanding-allocations-in-go-stack-heap-memory-9a2631b5035d) - benchmarking and analysing the difference between heap and stack variables
- [CPU profiler](https://www.jetbrains.com/help/go/cpu-profiler.html) - CPU profiling in GoLand IDE
- [When to use pointers in Go](https://medium.com/@meeusdylan/when-to-use-pointers-in-go-44c15fe04eac) - use cases for pointers and non-pointer variables
- [Go: Should I Use a Pointer instead of a Copy of my Struct?](https://medium.com/a-journey-with-go/go-should-i-use-a-pointer-instead-of-a-copy-of-my-struct-44b43b104963) - showing an actual use case in which heap variables are slower
