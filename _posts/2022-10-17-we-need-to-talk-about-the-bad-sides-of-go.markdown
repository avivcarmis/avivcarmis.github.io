---
medium: https://itnext.io/we-need-to-talk-about-the-bad-sides-of-go-568a1e5adbc6
date: 2022-10-17 11:11:42
title: We Need To Talk About The Bad Sides of Go
description: The downsides of the Go programming language
image: images/posts/we-need-to-talk-about-the-bad-sides-of-go/we-need-to-talk-about-the-bad-sides-of-go.png
categories: [go, programming-languages]
tags: [go, safety, naming, error-handling, concurrency]
series: goat
---
This is the second part of a 3-article series. This is a story about the downsides of the Go programming language, the part about it that makes us less productive and our codebases less safe and less maintainable. And about propositions for improvement. 🌟

#### More in this series
*   [What Makes Go the Best Language]({% post_url 2022-10-17-what-makes-go-the-best-language %})
*   **We Need To Talk About The Bad Sides of Go**
*   [A Proposition For a Better Future]({% post_url 2022-10-17-a-proposition-for-a-better-future-for-go %})

In the [intro to the previous article]({% post_url 2022-10-17-what-makes-go-the-best-language %}) we’ve presented a conflict — Go has an aggressive tendency to remain simple. Although it has tremendous upsides, it prevents Gophers from getting more productive. Then, we discussed the stronger sides of the language and everything that makes it unique. In this one, we’ll dive right into discussing and presenting the **problematic sides of Go**. If you’ve missed the [previous one]({% post_url 2022-10-17-what-makes-go-the-best-language %}), I suggest glancing back as it provides an important context for the full-on rant that’s coming your way. Ready? Let’s start.

## Namespace Pollution and Bad Naming

After writing Go for a while you start noticing you usually run out of reasonable variable names relatively quickly. Initially, it sounds like a cosmetic problem, but it’s not. Let’s start with why this happens.

### Lack of Visibility Modifiers

The first root cause of namespace pollution is the lack of visibility modifiers. In what I believe to be an effort to reduce redundant keywords and enhance simplicity, the language designers decided to omit visibility modifier keywords (`public`, `private`, etc…) in favor of symbol naming. Symbols starting with an uppercase letter are automatically public and the rest are private. Sounds like a great choice to promote simplicity. But over time it’s becoming clear that this method has a stronger downside than upside: In most other languages, type names, by convention, begin with an uppercase letter, and variable names begin with a lowercase one. This convention has a very powerful implication — it means that variables can never shadow types. Consider the following Go code:

```go
// type shadowing in Go

type user struct {
  name string
}

func main() {
  user := &user{name: "John"}
  anotherUser := &user{name: "Jane"} // compilation error: user is not a type
}
```

This is very common in Go, and I bet most Gophers run into this at some point. In most cases, your coding scopes deal with one user instance, so naming it `user` should be a clear and reasonable choice. However, in Go, whenever you store a private type into a private variable or a public type into a public variable — you run into this. **So you simply start naming your user variables** `u`.

### Package Scope Namespace

A second reason for namespace pollution, and probably the most annoying one — **package scope namespace**. Let’s start with an example:

```go
// file: client/slack_client.go
package client

const url = "slack.com"

type SlackClient struct {...}
```

```go
// file: client/telegram_client.go
package client

const url = "telegram.com" // compilation error: url redeclared in this package

type TelegramClient struct {...}
```

Why? Those are two separate files. Can’t I declare anything private and only use it locally? I can’t. Any symbol declared in a file is automatically visible to the entire package.

In some cases, it makes perfect sense for packages to share a private symbol between several files (i.e. [package-private visibility](https://idratherbewriting.com/java-access-modifiers/#public-private-protected-package-private)). In other cases, symbols would have no reason to escape the scope of their file. The fact that Go does not offer this kind of control **leads to heavily cluttered packages** and unnecessarily long and specific symbol names to avoid duplication and ambiguity. In big packages of big projects, it becomes next to impossible to find a reasonable name that is also free to use. Not to mention finding the actual function you wish to call out of a list of 100s or even 1000s of symbols declared within the package. 😭

### Builtin Symbols

Lastly, global built-in symbols. Let’s begin with an example:

```go
// shadowing builtin symbols

func larger(a, b []string) []string {
  len := len(a)
  if len > len(b) { // compilation error: invalid operation: cannot call non-function len (variable of type int)
     return a
  }
  return b
}
```

In other words, there’s a list of names you could, technically, use to name your variables, but if you do you **shadow important built-in functionality**.

A possible solution to this problem: built-in symbols should be keywords instead of symbols. It never makes good sense to override `len`, `make` or `append`, right? Let alone `true` and `false` (those are not keywords in Go). If we can agree this is always a bad practice, why permit it in the first place?

An even better solution: built-in symbols should be placed under contextually related namespaces. I.e. `len`, `append`, and `cap` would make a lot more sense as methods of slices than global symbols. They would all read more naturally, but more importantly, they would not clutter the global namespace, allowing us to safely use them as reasonable variable names when needed.

### Is It Really That Important?

Are variable names that important? Well, for starters, yes they are. A user variable named `u` is never a better idea than a well, contextually oriented variable name. But it also breaks the first rule of Go — **readability over writability**. Trying to figure out a code snippet consisting of `u`, `r`, `z`, and `t` variables is actually quite nightmarish. I get that there are cases where the code is short and simple enough, and the context is clear enough for a `u` variable to remain clear. But why are we [officially encouraging it](https://github.com/golang/go/wiki/CodeReviewComments#variable-names) in the first place? What’s the added value of `flname` over `fileName` in any modern environment, where source code size implications are negligible compared to its maintainability? Personally, I find myself constantly informing typo-check tools that “no actually conns is a valid word, it’s obviously an abbreviation for connections, and…what? cnt? no no lol it’s just short for count, naughty”. 🤦‍♀

The official Go wiki provides a section of best practices via code review comments. There’s one specific piece of advice to keep variable names short as [familiarity admits brevity](https://github.com/golang/go/wiki/CodeReviewComments#receiver-names). Being the not-native-English-speaker that I am, I thought “_Hmm…that sounds smart_”. 🤔 Took me a few more seconds to grasp this sentence for what it really means — “If it seems familiar when you write it, allow yourself to be a bit lazy”. 😮 I mean, but why? This is exactly the kind of convention that disguises itself as good practices and ends up encouraging us **in the wrong direction**. I’ve found myself on multiple occasions staring at hugely popular open source projects, wondering how can anyone make sense of what I’m seeing right now. Just to prove a point, [here](https://github.com/valyala/fasthttp/blob/16d30c474cea55710ff9a550f1b20b7b974053d9/client.go#L1425) is a line of code referencing `cc.c` in the middle of a 100+ line function, in the middle of a 2947 line file. I obviously can’t blame the author (yea I know, I _can_ git blaming them, ok shush) because they were simply following the rules of idiomatic Go. But figuring out the meaning of `cc` and `c` in such a huge function within such a huge file and package is definitely not productive programming.

Another problem with a low variety of variable names is accidental shadowing and accidental rewrites. Consider the following function:

```go
// an annoying little bug hides in this code

func (u *user) Approve() error {
 err := db.ApproveUser(u.ID)
 if err != nil {
   err = reportError("could not approve user")
   if err != nil {
     log.Error("could not report error")
   }
 }
 return err
}
```

Ouch, this one is an annoying little bug to figure out in production. Can you find it? It should be covered by a code review, but it takes a pretty thorough one to reveal it. How else can you prevent it? Make sure not to rewrite variable values when it’s not needed (which Go does not enforce. we’ll cover it shortly), and then strive to provide a **well contextually oriented name for each variable**. In some ways, this is the opposite of idiomatic Go, and reducing the variety of available names hurts it even further.

### Acronyms

Another annoying hit to readability is the uppercase acronyms convention. [The idiomatic way](https://github.com/golang/go/wiki/CodeReviewComments#initialisms) to name a public variable referring to a URL of some HTTPS endpoint would be `HTTPSURL`. Aside from the fact that it breaks typo-check tools, am I the only one who thinks that `HttpsUrl` is better in every possible way? What about when three of those are concatenated? Is `JSONHTTPSURL` seem like a good variable name to anyone? Feels like trying to figure out the name of a terrible Reddit sub. 😂🤷‍♀️

### Receiver Names

And lastly — method receiver name conventions. I suggest reading [Jesse Duffield’s](https://twitter.com/DuffieldJesse) beautifully written ‘Going Insane’ [piece about conventions](https://jesseduffield.com/Gos-Shortcomings-5/#receiver-names). I won’t be able to present or argue any better than that. In short: `self` or `this` would probably make a lot more sense than these [single-letter receiver names](https://github.com/golang/go/wiki/CodeReviewComments#receiver-names) we use and then find ourselves chasing our tails renaming whenever we rename the struct or move methods around.

## Value Safely

Much like type safety, value safety is a means for making runtime assurances and guarantees during compile time. While type safety refers to the types of values in runtime, value safety refers to the actual values. And while type safety is performed by a single, consistent typing system, value safety is merely a set of different tools and features that provide various guarantees regarding values. Due to that, languages can do more than simply opting in or out of value safety in its entirety. Rather, each language chooses the exact set of value safety features to support and enforce.

Some value safety tools are actually well-known basic concepts of programming that either provide safety at their basic essence or as a side effect. Others are exciting modern ones that evolved over time. One way or the other, value safety features are not productivity-enhancing syntactic-sugary tools that modern language enthusiasts brag about. They provide highly important guarantees we critically rely on. Unlike syntactic sugar features, these guarantees cannot be achieved by writing explicit code. In their absence, all that’s left is human responsibility and our ability to analyze edge cases and pitfalls of each piece of code (LOL? 🤣). We can choose to opt-out of that as well, and things will keep on working as long as everybody succeeds in writing code exactly as intended. Until someone doesn’t, and then it’s time to panic. Literally.

As our understanding of programming principles evolves, new tools provide better value safety. Some modern languages provide modern, exciting new ways to enforce value safety, allowing programmers to write more **robust and performant code**. Other compiled languages provide at least some value safety features. **Go doesn’t provide any**. Now it’s on us to make sure we use everything as expected. This is a scary situation both as a consumer and a producer of any library or piece of code. Let’s address those missing features.

### Null Pointer Safety

Definitely not the worst design decision in programming history, but probably [the most notorious one (“the billion dollar mistake”)](https://en.m.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions) — null pointer exceptions cost too much money to too many people regularly. At the end of the day, the cause for this is always the simplest human mistake — **forgetting to check for edge cases**. This ever-recurring problem acts as a constant reminder that we’re simply not made for this. In programming, “_please remember to do this and that_’’ is always a bad sign. If there’s one thing you can count on — we won’t. We’re bad at remembering to do stuff.

Modern language features allow for eliminating this problem completely. **Strict null checks** (like the features provided by [Swift](https://developer.apple.com/documentation/swift/optional), [Kotlin](https://kotlinlang.org/docs/null-safety.html), and [TypeScript](https://www.typescriptlang.org/tsconfig#strictNullChecks)) allow us to explicitly define when to permit null values. When permitted, null checks are required before dereferencing them. Other languages provide **sum types (**like the ones in [Rust](https://www.rust-lang.org/) and [Scala](https://www.scala-lang.org/api/2.13.x/scala/Option.html)) that prevent null values in the first place.

For the life of me, I cannot understand why a programmer would advocate type safety and neglect null safety. If you want your compiler to enforce not calling an undefined method, wouldn’t the same logic apply when you try to dereference a null value? I believe, one way or the other, modern languages must be equipped with such an enforcement mechanism.

[Here’s a proposal](https://github.com/golang/go/issues/49202) to add nullable-type support to Go.

## Enums

It doesn’t matter how you model your code, eventually, you’re always going to end up needing enumerations. A variable that can have one out of several possible values. If you only have two possible values — booleans will work great. If you have three options or more, you will need a mechanism to support enumerations.

In Go, you do…but not really. You can define constants, but that’s all they are — simple constant values. **They do not guarantee a hosting variable to necessarily hold one of them**. Even if you define a custom type. Here’s an example:

```go
// enum with an invalid value

type ConnectionStatus int

const (
  Idle ConnectionStatus = iota
  Connecting
  Ready
)

func main() {
  var status ConnectionStatus = 46 // no compilation error
}
```

The value of the `status` variables declared in the main function should either be 0, 1, or 2. Yet, no compilation or runtime error occurs when we set it to 46. A good developer should not set it to 46, duh! 🤪 But mistakes do happen. And as the code base and the amount of engineers scale, it’s only a matter of time until they do. A good compiler should help us avoid it.
Don’t forget that 46 may also be accepted from external, non-hard-coded sources. Like HTTP requests or files, for example. We should always **remember** to manually validate it. Until we forget. 😞

Another problem with this approach is the lack of encapsulation. Enum-related behavior cannot be defined within the enum itself but only via switch casing. However, switch cases in Go do not enforce going over all possible values. Here’s what it looks like:

```go
// unhandled enum case

type ConnectionStatus int

const (
  Idle ConnectionStatus = iota
  Connecting
  Ready
  Disconnecting
)

func handleConnectionStatus(status ConnectionStatus) {
  switch status {
  case Idle:
     reconnect()
  case Connecting:
     wait()
  case Ready:
     useConnection()
  }
}
```

Doesn’t seem that bad, doe’s it? But it is — each time you add a new enum value you have to **search your codebase** for those switch cases. If you missed one, like the example above fails to properly handle `Disconnecting` status, it falls into undefined behavior. And rest assured, you will always remember to fix those enums, until one day you don’t (remember what we just said about remembering?…you probably don’t. Me too 🤦‍♀️). Can you imagine a scenario where you prefer not to have compiler enforcement for this kind of pitfall?
Solutions to this problem are simple: either you require switch case statements to exhaust all possible values ([as rust does](https://doc.rust-lang.org/book/ch06-02-match.html#matches-are-exhaustive), for example), or you require enums to implement behavior ([as java does](https://jenkov.com/tutorials/java/enums.html#enum-abstract-methods), for example).

Other problems with lack of native enum support? What about **iterating all possible values** of an enum? That’s something you need every now and then, maybe if you need to send it to the UI for a user to choose one. Not possible. What about **namespacing**? Isn’t it better for all HTTP status codes to reside in a single namespace providing only HTTP status codes? Instead, Go has them mixed with the rest of the public symbols of an HTTP package — like Client, or Server, and error instances.

[Here’s one of the enum proposals](https://github.com/golang/go/issues/28987) in Go.

### Struct Default Values

In some cases, structs may be more than simply a collection of variables, but rather, a concise entity with state and behavior. In such cases, it may be required for some fields to initially hold meaningful values and not simply their zero values. We might need to initialize int values to -1 rather than 0, or to 18, or to a calculated value derived from other values.

Go does not provide any realistic approach to **enforce initial state in structs**. The only way to sort of achieving this is by declaring a public constructor function and then making your struct private to prevent direct instantiation via [struct literals](https://go.dev/tour/moretypes/5#:~:text=A%20struct%20literal%20denotes%20a,pointer%20to%20the%20struct%20value.). At this point, you have to declare an interface describing the struct’s public methods to be able to export the return value of the constructor. Example:

```go
// trying to enforce initial values in structs

type connectionPool struct {
  connectionCount int // initial value should be 5
}

type ConnectionPool interface {
  AddConnection()
  RemoveConnection()
}

func newConnectionPool() ConnectionPool {
  return &connectionPool{connectionCount: 5} // since the struct is private, direct instantiation can only be done from inside the package
}

func (c *connectionPool) AddConnection() {
  c.connectionCount++
}

func (c *connectionPool) RemoveConnection() {
  c.connectionCount--
}
```

Since the struct itself is not exported, it’s not possible to directly instantiate it via struct literals, but rather, only via a call to our constructor. This will make sure we enforce the desired initial value.

This approach might barely be worth the effort when maintaining open source libraries. But what about us, mortal beings? We do not care so much about the integrity of the API of some structs internal to our process, **we’re simply trying to quickly fix a bug in production**. If we work like this, every time we make a change to a method signature we have to also change the duplication in the interface. What a random side effect for simply wanting to enforce initial values in our struct 🤦‍♀️😂
But that’s not all. It doesn’t even completely solve the problem — what about code within the same package, it can still use struct literal and **skip the constructor** and spoil everything we’ve tried to accomplish. 🤤

Another problematic use case is config structs, but the best way to showcase it is with a real example. [Sarama](https://github.com/Shopify/sarama) is the most popular Go library for Kafka. A huge, mature project, maintained by Shopify. Sarama exposes a [Config](https://github.com/Shopify/sarama/blob/1cbdb49dd3be553e96301d185603b5ab7bee1005/config.go#L21) struct which should be passed in on the creation of new clients. It contains various information on how to connect to Kafka brokers and how to maintain connection state and behavior. The struct comments indicate the intent of each field and its default value. For example — the [Max field here](https://github.com/Shopify/sarama/blob/1cbdb49dd3be553e96301d185603b5ab7bee1005/config.go#L27), which sets the max retry times of a request to Kafka and defaults to 5. So an average developer, like me for instance, may think “_hmmm…okay, let’s pass in an empty struct literal to use the default value of 5_”. And so I did.

```go
// using sarama config with default values

func createConsumer(brokers []string) {
  consumer, err := sarama.NewConsumer(brokers, &sarama.Config{})
  // ...
}
```

“_…Now my config surely allows 5 retries,_” I thought to myself. 🤗 It was only until a couple of days later when I played with it a bit more and thought to myself “_but wait, how could they know?_”. How can Sarama code be able to distinguish between a default value of 0 passed in using `&sarama.Config{}` which should allow 5 retries, and a 0 explicitly passed in using `&sarama.Config{Max: 0}` which should allow 0 retries. 😨
Indeed, it can’t. Apparently, there’s a [constructor for the config struct](https://github.com/Shopify/sarama/blob/1cbdb49dd3be553e96301d185603b5ab7bee1005/config.go#L454) that I should have used instead of directly instantiating it. LOL 🤦‍♀️. “But wait a minute,” I thought to myself, “could I have pushed code instructing Sarama library to use 0 retries **and zero values to all other config params**??!” 😲😧😨😱 Yup. I most definitely did. 😩🤦‍♀️ 🤦🏽‍♂️

Initially, I was furious with the designers of the library for not enforcing prevention of such a mistake, “if it happened to me” and everything. Then I thought about it more and realized, they can’t. Go simply does not provide it. At glance, struct literals look like the perfect candidate for config params use case, allowing you to pass in exactly what you need, and omit the rest. But turns out it’s the opposite.

[Here’s a proposal](https://github.com/golang/go/issues/32076) to support initializers for types.

### Const Assignments

After spending some time with languages that distinguish between constant and variable values, you start noticing that the vast majority of your assignments are constant. Non-constant assignments are mostly used during calculation that requires counting, concatenation, or other forms of aggregation. **In most other use cases, assignment to a variable is a single-time operation**. Let’s take [reactjs GitHub repository](https://github.com/facebook/react), for example, comparing the amount of `let` keyword usage to the amount of `const` keyword usage we can conclude that **constant assignments constitute 75% of all assignments** (25,440 vs 6,815). Therefore, it would make a lot of sense to treat assignments as constant by default, as some other modern languages ([like rust](https://doc.rust-lang.org/std/keyword.mut.html)) do. Why is it helpful? Let’s get back to the example from variable naming, where we talked about accidental shadowing and rewriting variables:

```go
// shadowing variables may lead to implicit behavior

func (u *user) Approve() error {
 err := db.ApproveUser(u.ID)
 if err != nil {
   err = reportError("could not approve user")
   if err != nil {
     log.Error("could not report error")
   }
 }
 return err
}
```

Remember this one? The bug here is that the second assignment to the `err` variable may clear up the original error, causing us to **return from the function like no error ever occurred**. This would cause a calling function to **keep executing as if the user has actually been approved, although they weren’t.** An experienced Gopher would probably find it rather quickly if they know they should look for a bug. An inexperienced Gopher would probably take a lot longer. **Constant assignments would have prevented it easily**. The second assignment to the error variable would not have compiled, instructing us to define a separate variable.

Constant assignments are not only about compiler enforcement. They’re also about the **author being able to convey intent**, be more expressive, and make the code clearer. If you know some variable was not meant to be reassigned, you have **more information coming to modify or refactor** the code surrounding it. ℹ️🤓

[Here’s a const assignment support](https://github.com/golang/go/issues/6386) proposal.

### Immutability

Immutable values are a separate thing. An assignment to a variable can be constant, yet **the value itself can still mutate**. Consider creating a constant variable pointing to a map, and then perform a write operation on the map. The variable is still pointing to the same map, but the map value itself has changed.

Immutable data structures can be a tremendous tool to prevent data races in concurrent environments. In Go, there’s **no native support for immutable data structures**. The good news is with the arrival of generics we can create them ourselves. For example `ImmutableMap[K, V]`.

[Here’s a proposal for immutability](https://github.com/golang/go/issues/27975).

## Error Handling

Error handling is probably the biggest debate in the Go community, now that the generics debate is all settled. Over time, I’ve gotten myself familiar with many approaches for error handling, either in other languages or in Go proposals. I’m far from arguing that I can come up with the best mechanism for error handling myself. I do want to mention that Go error handling has many strengths and advantages. However, we’re here to talk about the downsides. First, I want to refer (again) to Jesse Duffield’s [going insane piece about error handling](https://jesseduffield.com/Gos-Shortcomings-1/), beautifully laying out his own pain points. Then I’ll add two points myself.

First, I’m hearing a lot of Gophers lately arguing that the explicit error handling approach in Go is a good idea since it forces us to deal with every error. That we should never simply propagate errors. In other words, **this should never happen**:

```go
// error handling without wrapping
if err != nil {
  return err
}
```

Instead, we should always wrap errors with new errors to provide more context for the caller, like this:

```go
// error handling with wrapping
if err != nil {
  return errors.Wrap(err, "read failed")
}
```

This argument does not sit with me **at all**. One of the first arguments I heard in favor of Go’s error handling mechanism was that it’s **much more performance than try-catch-based mechanisms**. I tried it out, and it turns out to be dramatically true. One of the main reasons for that low performance in try-catch environments is the creation of the full stack trace and exception information in every level of the call stack whenever an exception is thrown. In Go, wrapping each error with another error up the entire call stack, and then garbage collecting all the created objects is almost as expensive. Not to mention the manual coding. If this is what you’re advocating, you’d better off in a try-catch environment, to begin with.

Stacked traces can be very useful when investigating errors and bugs, **but the rest of the time they’re highly expensive redundant information**. Maybe a perfect error-handling mechanism would have a switch to turn them on and off when needed. 🤷‍♀️

The second point actually has nothing to do with the error handling mechanism itself, but rather, with conventions. We’ve discussed error handling performance in the previous paragraph. [Sentinel errors](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully#sentinel%20errors) give Go another performance bump for error handling. I suggest reading more about them, but in short, when all occurrences of some error contain the same information, you can simply create a single error object instance instead of generating new ones on each occurrence. Singleton objects to represent errors. We call them **sentinel errors**. They prevent **unnecessary allocations** and **garbage collection**. In addition to the performance bump, error comparison becomes simpler. Since those error values are singletons, instead of comparing their type you can simply compare their value using a simple equality operator.

```go
// sentinel error
var NetworkError = errors.New("oops...network error")

func IsNetworkError(err error) bool {
 return err == NetworkError
}
```

Notice how this type of comparison doesn’t require type checking. This works great — it reads and writes well, and also performs well. 👏🤝

Sometimes, however, sentinel errors **can’t** be used. Sometimes we want the error message to contain information relevant to any **specific occurrence of the error**. (i.e. `"oops...network error <root cause>"`). This means we must instantiate a new error object each time. At this point, due to the lack of solid conventions, we simply go:

```go
// lazying out when creating non sentinel errors
return errors.New("oops...network error " + rootCause)
// or
return fmt.Errorf("oops...network error %s", rootCause)
```

Oops indeed. 🤦‍♀️ Since there’s **no specific type or instance to this error**, we’ve now lost any ability to effectively check for this type of error.

```go
// how can we distinguish network errors?

err := doSomething()
if /* err is network error */ { // 🤔 how can we tell?
  // handle network error
} else if err != nil {
  // handle other errors
}
```

Our only option is to check `strings.Contains` on the error message itself:

```go
// our only option for error handling is terrible

err := doSomething()
if strings.Contains("oops...network error") {
  // handle network error
} else if err != nil {
  // handle other errors
}
```

This is terrible in terms of performance. But more scary is the fact that it’s not guaranteed. Obviously, `"oops...network error"` can change at any time, and there isn’t any compiler enforcement to help us out. When the author of the package decides to change the message to `"oops...there has been a network error"`, **my error handling logic breaks**, you’ve gotta be kidding me. 🤷‍♀️🤦‍♀️
You _can_ do it in any other language as well. In Java, for example, you could `throw new Exception("oops...network error")`. But it will most probably not pass code review for internal code in a small startup company. In Go, however, it passes code reviews in huge open-source libraries maintained by huge organizations (how about [google’s protobuf](https://github.com/protocolbuffers/protobuf-go/blob/v1.28.0/internal/errors/errors.go#L84)). I personally found myself falling back to string contains checks, feeling sick to my stomach with more than one major open-source library. 🤢🤒

Either by a strong, solid convention or by compiler enforcement: `errors.New` and `fmt.Errorf` **should only be used to create sentinel errors**. Any other error returned must declare a dedicated, exported type to allow for reasonable handling. As authors of libraries, if we neglect to do so, we risk the code safety and integrity of our consumers.

[Here’s a language proposal](https://github.com/golang/go/issues/27567) to add a `?` operator for error handling. It’s worth mentioning that this proposal closed in 2018 due to overheated discussion and never opened back. I guess some things are better left undealt with. 😂

## Async Return Values

There are many synchronization mechanisms in Go, some of them are native and some are provided by the Go SDK. Others are available in many open source libraries. Despite that, you could say that Go code is usually dominated by **two main synchronization mechanisms** — channels (native) and `WaitGroups` (provided by the SDK). Those are two powerful mechanisms that can support the implementation of every possible concurrent flow. `WaitGroups` allow synchronization of execution timing of different threads, and channels allow both such synchronization and passing values between threads.

Each mechanism has its set of go-to use cases, but there’s yet another use case left uncovered, and I argue that this third use case is, oftentimes, the best practice. To see it in action, let’s consider this very popular example: **we want to fetch several resources concurrently, and then combine the results**. Let’s first implement it with `WaitGroups`.

```go
// first attempt: using sync.WaitGroup

func fetchResourcesFromURLs(urls []string) ([]string, error) {
  var wg sync.WaitGroup
  var lock sync.Mutex
  var firstError error
  result := make([]string, len(urls))
  for i, url := range urls {
     wg.Add(1)
     go func(i int, url string) {
        resource, err := fetchResource(url)
        result[i] = resource
        if err != nil {
           lock.Lock()
           if firstError == nil {
              firstError = err
           }
           lock.Unlock()
        }
        wg.Done()
     }(i, url)
  }
  wg.Wait()
  if firstError != nil {
     return nil, firstError
  }
  return result, nil
}

func fetchResource(url string) (string, error) {
  // some I/O operation...
}
```

It’s a bit explicit, like Go usually is, but it works. The fact that we have to handle the errors ourselves did seem too explicit to the Go team, so they released an additional variation of `WaitGroup` called `ErrGroup`, which simplifies error handling:

```go
// second attempt: using errgroup.Group

func fetchResourcesFromURLs(urls []string) ([]string, error) {
  var group errgroup.Group
  result := make([]string, len(urls))
  for i, url := range urls {
     func(i int, url string) {
        group.Go(func() error {
           resource, err := fetchResource(url)
           if err != nil {
              return err
           }
           result[i] = resource
           return nil
        })
     }(i, url)
  }
  err := group.Wait()
  if err != nil {
     return nil, err
  }
  return result, nil
}

func fetchResource(url string) (string, error) {
  // some I/O operation...
}
```

`ErrGroups` aggregate errors and simplify the error handling, and also, they support `contexts` to allow unified timeouts and cancellation, and a concurrency limit.
Obviously, this is a more sophisticated synchronization mechanism, however, it still has two significant downsides: **we still have to somehow synchronize the return values**, and due to the signature of [ErrGroup.Go](https://pkg.go.dev/golang.org/x/sync/errgroup#Group.Go) function we have to **wrap our concurrent function with a function that receives no parameters**. And since Go doesn’t support shortened lambda expressions ([active proposal here](https://github.com/golang/go/issues/21498)) it becomes even more explicit and less readable.

The first downside described above can and should be solved with generics, the second one remains. This is a perfect mechanism for cases where we **do not accept and do not return anything in the concurrent functions**, but these cases are very rare.

Let’s move on to channels:

```go
// third attempt: using channels

func fetchResourcesFromURLs(urls []string) ([]string, error) {
  ch := make(chan *singleResult, len(urls))
  for _, url := range urls {
     go func(url string) {
        resource, err := fetchResource(url)
        ch <- &singleResult{
           resource: resource,
           err:       err,
        }
     }(url)
  }
  result := make([]string, len(urls))
  for i := 0; i < len(urls); i++ {
     res := <-ch
     if res.err != nil {
        return nil, res.err
     }
     result[i] = res.resource
  }
  return result, nil
}

type singleResult struct {
  resource string
  err       error
}

func fetchResource(url string) (string, error) {
  // some I/O operation...
}
```

Still, pretty explicit for such a common use case. **A lot of room to make mistakes that lead to race conditions and deadlocks**.

Now imagine if the `go` keyword not only spawns a new goroutine but also returns an object to track it, allowing us to **wait for the called function and get its returned values**. Let’s refer to it as a `promise[T]`.

```go
// imaginary promise syntax

func fetchResourcesFromURLs(urls []string) ([]string, error) {
  all := make([]promise[string], len(urls))
  for i, url := range urls {
     all[i] = go fetchResource(url)
  }
  var result []string
  for _, p := range all {
     resource, err := p.Wait()
     if err != nil {
        return nil, err
     }
     result = append(result, resource)
  }
  return result, nil
}

func fetchResource(url string) (string, error) {
  // some I/O operation...
}
```

Ahh, better. These kinds of objects are usually called `futures` or `promises` and most popular languages support them. In `all[i] = go fetchResource(url)` we populate a slice of promises, each one tracks the execution and return values of a different goroutine. Then we wait for all of them and fail in case of an error. (There’s usually [native support](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) for such an operation).

This last code snippet is imaginary. It **doesn’t exist in Go**. But if it would, it wouldn’t only be easier on the eyes, but it would also be safer. Since `WaitGroups` and channels do not work with return values, promise-like mechanisms have **2 main advantages**:

First, `WaitGroups` and channels require some implementation to synchronize the results. With `WaitGroups` we can find ourselves passing pointers around or using sharded slices like in the example above. Sometimes we would require an additional mutex. With channels, we have to create a buffered channel. Wrong buffer size and we deadlock. Either way, there’s a **potential risk** that would have been **avoided by using a promise-like mechanism**.

The second advantage lies within **pure-function programming**. Pure functions use return values instead of producing side effects. This means we can rely on the compiler to **ensure we do not create deadlocks**. While working with channels, for example, we have to receive a channel argument and explicitly call it with a result. For example:

```go
// can’t write pure functions when dealing with explicit synchronization mechanisms

func fetchResources(url string, ch chan *singleResult) {
  data, err := httpGet(url)
  ch <- &singleResult{
     resources: data,
     err:       err,
  }
}
```

Six months later, another engineer (it’s never us, obviously, always another engineer 🤨) can easily make the mistake of adding an if statement that returns, **forgetting to explicitly call the channel**. This **results in a deadlock**. No compilation error here, obviously:

```go
// non-pure functions are for concurrency are deadlock traps

func fetchResources(url string, ch chan *singleResult) {
  if url == "" {
     return
  }
  data, err := httpGet(url)
  ch <- &singleResult{
     resources: data,
     err:       err,
  }
}
```

With pure functions, you must specify return values when branching out the control flow. In other words, **the other engineer simply can’t deadlock**:

```go
// not possible to deadlock using pure functions

func fetchResources(url string) (string, error) {
  if url == "" {
     return // compilation error: not enough arguments to return
  }
  data, err := httpGet(url)
  return data, err
}
```

Additionally, the pure function form is **fully reusable**. This is simply a function that performs the operation and returns a result. There’s no familiarity with any specific synchronization mechanism. It does not receive channels or `WaitGroups` nor does it perform any explicit synchronization. Go simply takes care of everything else.

In my opinion, the lack of solid community conventions in combination with an ineffective async return value mechanism is the **root cause of terrible coding**, and this is kind of a **standard in the Go community.** \[my deepest and sincere apologies to everyone who’s hurt by this paragraph\]. Some examples? How about a [400-line function](https://github.com/valyala/fasthttp/blob/v1.38.0/server.go#L2055) in one of the most popular HTTP frameworks in Go 😨, how about a [100-line function](https://github.com/grpc/grpc-go/blob/v1.47.0/server.go#L737) in Google’s gRPC library? What about a [66-line function](https://github.com/mongodb/mongo-go-driver/blob/master/mongo/session.go#L178) with a nested 2-level while-true loop and a go-to statement inside it in the official MongoDB driver?! 😵‍💫😵‍💫😵‍💫

Those are just the first examples popping out of a quick google search. See what they all have in common? They combine complex for-loops or switch cases with `defer` and `go func` statements. In other words, since synchronizing async return values in Go requires passing around pointers and creating mutexes, it’s easier to write everything within one big function and avoid passing them around by capturing them in the [closure of nested lambda functions](https://yourbasic.org/golang/anonymous-function-literal-lambda-closure/). It sounds like the main problem here is maintaining those lengthy functions, but apparently, closure captured variables inside `go func` statements are also the **number one reason for data races in Go code**, according to a [very interesting study](https://eng.uber.com/data-race-patterns-in-go/) done at Uber engineering.

`WaitGroups` and `ErrGroups` are a great choice when performing **async or concurrent operations that do not return any value**. Channels are a perfect choice when dealing with **consumer-producer** use cases or waiting for several concurrent events using **select statements**. However, since most use cases of async calls produce return values and require error handling, my guess is if Go supported a promise-like mechanism, it **would have been the most popular choice** of Gophers. But I hope I’ve shown that for such use cases, it’s **also the safest one**.

Here are some interesting proposals ([1](https://github.com/golang/go/issues/22293), [2](https://github.com/golang/go/issues/17466)) to add such mechanisms to the language. The latter proposes something of aliasing channels to futures. Honestly, I couldn’t care less what we call them. I even like the idea of reusing channels instead of introducing a new concept. I simply want the `go` keyword to return an object allowing me to **track the return values of the executed function**.

## Summary

If you combine community conventions and naming problems with async return value problems, you end up with **hugely popular libraries** shipping code with complex, 100+ line functions, using one-letter undocumented variables, declared at the other side of the package. This is extremely **unreadable and unmaintainable, and surprisingly common**. In addition, as opposed to other modern languages, Go doesn’t provide any kind of runtime value safety. This results in many **value-related runtime issues** which can easily be avoided.

In the [last article of the series]({% post_url 2022-10-17-a-proposition-for-a-better-future-for-go %}), we’re going to discuss what we can do to improve it and a proposition for a better future for Go.
