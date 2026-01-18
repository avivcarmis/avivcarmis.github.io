---
date: 2026-01-17 12:00:00
title: "Go Fundamentals: Variable Behavior"
description: Understanding variable type behavior, zero values, and common pitfalls in Go
image: images/posts/go-fundamentals/go-fundamentals-variable-behavior.png
categories: [go]
tags: [go, fundamentals, variables, pointers]
series: go-fundamentals
---

This post covers Go variable behavior, including mutability, function pass types, zero values, and common pitfalls to avoid.

<div class="series-nav">
  <div class="series-nav__title">Go Fundamentals Series</div>
  <div class="series-nav__subtitle">Article 1 of 4</div>
  <div class="series-nav__list">
    <span class="series-nav__item--current">1. Variable Behavior ← You are here</span>
    <a href="/go-fundamentals-memory-architecture" class="series-nav__item">2. Memory Architecture</a>
    <a href="/go-fundamentals-concurrency-model" class="series-nav__item">3. Concurrency Model</a>
    <a href="/go-fundamentals-error-handling" class="series-nav__item">4. Error Handling</a>
  </div>
</div>

## Variable Type Behavior

### Mutability

What will the following python code print?

```python
def func1():
    x = 1
    func2(x)
    print(x)

def func2(x):
    x += 1

func1()
```

<details>
<summary>Click to reveal answer</summary>

The answer is `1`. Integers are immutable in Python, so `x += 1` in `func2` creates a new local variable rather than modifying the original.

</details>
<br/>

What will the following python code print?

```python
def func1():
    x = {"key": 1}
    func2(x)
    print(x["key"])

def func2(x):
    x["key"] += 1

func1()
```

<details>
<summary>Click to reveal answer</summary>

The answer is `2`. Dictionaries are mutable in Python, so the modification in `func2` affects the original dictionary.

</details>

### Primitive Vs. Non-Primitive

If you've programmed in either Python, JavaScript, Java, C#, PHP and many more languages, this should feel very intuitive:

- Primitive variables are immutable.
- Non primitive variables are mutable.

Why? Function pass types.

## Function Pass Types

### Pass By Value

![Pass By Value](/images/posts/go-fundamentals/pass-by-value.png)

In pass by value:
- Two copies of `count` are created
- Caller and callee point to two different copies of the variable `count`
- Changes in `count` are made locally
- The value of `count` is not updated after the function call

### Pass By Reference

![Pass By Reference](/images/posts/go-fundamentals/pass-by-reference.png)

In pass by reference:
- Caller and callee point to the same variable `count`
- Variable `count` is updated inside the function call
- The value of `count` is updated after the function call

## Go Function Pass Types

### Pass By Value

```go
func main() {
    i := 1
    f(i)
}

func f(i int) {
    i++
}
```

In Go, every variable type can be passed by reference if it's a pointer.

### Pass By Pointer

```go
func main() {
    i := 1
    f(&i)
}

func f(i *int) {
    *i++
}
```

## Zero Values

### Nullability

Only the following types can be `nil`:

- pointer types
- map types
- slice types
- function types
- channel types
- interface types

As opposed to other languages: **structs** cannot be nil unless they are pointers, and **"primitives"** can be nil if they are pointers. Due to that, Go does not have the notion of primitive variables.

### Zero Values

What will be the value of `someVariable`?

```go
func main() {
    var someVariable int
    fmt.Println(someVariable)
}
```

Zero value by type:

```markdown
| Type                     | Value                                                                       |
|--------------------------|-----------------------------------------------------------------------------|
| Numeric variables        | `0`                                                                         |
| Strings                  | `""`                                                                        |
| Booleans                 | `false`                                                                     |
| Structs                  | instance of the struct with all fields initialized to their own zero values |
| Arrays (but not slices)  | all elements inside initialized to their own zero values                    |
| Everything else          | `nil`                                                                       |
```

### Struct Zero Value

What will be the zero value of the following struct:

```go
type Parent struct {
    a int
    b string
    c bool
    d child
}

type child struct {
    a float64
    b chan bool
}

func main() {
    var someVariable Parent
    fmt.Println(someVariable)
}
```

<details>
<summary>Click to reveal answer</summary>
<div markdown="1">

Output: `{0  false {0 <nil>}}`

- `a int` → `0`
- `b string` → `""` (empty string, appears as space in output)
- `c bool` → `false`
- `d child` → `{0 <nil>}` (float64 is `0`, chan bool is `nil`)

</div>
</details>

## Variable Behavior

### Behavior of Arrays

Arrays have fixed size:

```go
func main() {
    var x [4]int
    x[1] = 1
    fmt.Println(x) // [0 1 0 0]
}
```

### Behavior of Slices

Slices have **dynamic size** but they use a **backing array**. Slices can be seen as **pointers to their backing array**.

```go
func main() {
    x := []int{1, 2, 3, 4}
    y := x[1:]
    fmt.Println(y) // [2 3 4]
    mutateSlice(y)
    fmt.Println(x) // [1 2 14 4]
}

func mutateSlice(y []int) {
    y[1] = 14
}
```

Slices have `len` and `cap`, where:

- `len` is the length of the slice and the last relevant element in the backing array
- `cap` is the current capacity of the slice and size of the backing array

```go
func main() {
    result := make([]int, 2, 4)
    result[0] = 1
    result[1] = 2
    fmt.Println(result) // slice is [1 2]
    // backing array is [1, 2, 0, 0]
}
```

What will happen here?

```go
func main() {
    result := make([]int, 2, 4)
    result[0] = 1
    result[1] = 2
    fmt.Println(result[2])
}
```

<details>
<summary>Click to reveal answer</summary>
panic: runtime error: index out of range [2] with length 2

goroutine 1 [running]:
main.main()
Process finished with the exit code 2
</details>

#### Creating a new slice

```go
func appendSuffixToSliceElements(slice []string, suffix string) []string {
    result := []string{}
    for _, element := range slice {
        result = append(result, element + suffix)
    }
    return result
}
```

Empty slices behave like `nil` (**with the exception of json**), and they perform better.

This is a bit better, but still has bad performance (append has `O(n)` complexity):

```go
func appendSuffixToSliceElements(slice []string, suffix string) []string {
    var result []string
    for _, element := range slice {
        result = append(result, element + suffix)
    }
    return result
}
```

Improve it by creating the backing array with the required capacity when possible:

```go
func appendSuffixToSliceElements(slice []string, suffix string) []string {
    result := make([]string, 0, len(slice))
    for _, element := range slice {
        result = append(result, element + suffix)
    }
    return result
}
```

But note the difference between `len` and `cap`.

What happens if you specify len instead of cap?

```go
func main() {
    slice := []int{1, 2, 3}
    result := make([]int, len(slice))
    for _, i := range slice {
        result = append(result, i)
    }
    fmt.Println(result) // [0 0 0 1 2 3]
}
```

vs.

```go
func main() {
    slice := []int{1, 2, 3}
    result := make([]int, 0, len(slice))
    for _, i := range slice {
        result = append(result, i)
    }
    fmt.Println(result) // [1 2 3]
}
```

### Behavior of Maps

Maps are always effectively pointers:

```go
func main() {
    x := map[string]interface{}{"a": 1}
    fmt.Println(x) // map[a:1]
    mutateMap(x)
    fmt.Println(x) // map[a:false]
}

func mutateMap(x map[string]interface{}) {
    x["a"] = false
}
```

## Pitfalls & Code Smells

### Pointer to non-structs

Pointer to non structs *can be considered* as **code smells**. Usually, they can be refactored to a function that takes a value and returns a value, instead of a mutating function. (pure functions, no side effects).

```go
func main() {
    x := 3
    y := 4
    add(&x, y)
    fmt.Println(x) // 7
}

func add(x *int, y int) {
    *x = *x + y
}
```

Pure functions are:

- More readable
- Easier to reason about
- Easier to combine
- Easier to test
- Easier to debug
- Easier to parallelize

Better approach:

```go
func main() {
    x := 3
    y := 4
    x = add(x, y)
    fmt.Println(x) // 7
}

func add(x int, y int) int {
    return x + y
}
```

### Pointers to interfaces

**Pointers to interfaces** will not work as you expect:

```go
type SomeInterface interface {
    Foo()
}

func main() {
    var x *SomeInterface
    x.Foo() // Unresolved reference 'Foo'
}
```

### The nil interface pitfall

What will this function print:

```go
func main() {
    err := validateAge(19)
    if err != nil {
        fmt.Println("you are younger than 18")
    } else {
        fmt.Println("you are older than 18")
    }
}

func validateAge(age int) error {
    var err *TooYoungError = nil
    if age < 18 {
        err = &TooYoungError{}
    }
    return err
}

type TooYoungError struct {}

func (t *TooYoungError) Error() string { return "too young" }
```

Output:
```
you are younger than 18
```

In Go, `error` is a **builtin** interface:

```go
type error interface {
    Error() string
}
```

To understand what happens, let's look at Go's interface variable **memory layout**:

![Interface Memory Layout](/images/posts/go-fundamentals/interface-memory-layout.png)

An interface variable contains two parts:
- `type` - pointing to the concrete type
- `value` - pointing to the actual value

Meaning, if we set a concrete type to an **interface variable**, even if the actual value is `nil`, it will be a non-nil interface pointer, which points to a `nil` value.

```go
func main() {
    var x *SomeImplementation = nil
    var y SomeInterface = x
    fmt.Println(y) // <nil>
    fmt.Println(y == nil) // false
}

type SomeInterface interface {
    Foo()
}

type SomeImplementation struct {}

func (s *SomeImplementation) Foo() {}
```

![Interface with nil value](/images/posts/go-fundamentals/interface-nil-value.png)

How can we **avoid it**? Always declare error variables as `error`.

```go
func validateAge(age int) error {
    var err error
    if age < 18 {
        err = &TooYoungError{}
    }
    return err
}
```

Output:
```
you are older than 18
```

### For loop variable capture

Let's discuss for loop behavior: what will be the output of the following code? Also, what is the `Sleep` call for?

```go
func main() {
    x := []int{1, 2, 3, 4}
    for _, i := range x {
        go func() {
            fmt.Println(i)
        }()
    }
    time.Sleep(time.Second)
}
```

Output:
```
4
4
4
4
```

How to fix it?

Option 1 - Pass as parameter:

```go
func main() {
    x := []int{1, 2, 3, 4}
    for _, i := range x {
        go func(i int) {
            fmt.Println(i)
        }(i)
    }
    time.Sleep(time.Second)
}
```

Option 2 - Create a new variable:

```go
func main() {
    x := []int{1, 2, 3, 4}
    for _, i := range x {
        i := i
        go func() {
            fmt.Println(i)
        }()
    }
    time.Sleep(time.Second)
}
```

As of Go 1.22, this is no longer an issue. However, you'll still see many of these in many Go codebases.
