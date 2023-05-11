---
date: 2022-10-17 11:13:09
title: A Proposition For a Better Future For Go
description: An option for Go to move forwards with
image: images/posts/a-proposition-for-a-better-future-for-go/a-proposition-for-a-better-future-for-go.webp
categories: [go, programming-languages]
tags: [go, goat]
series: goat
---
This is a piece concluding a 3-article series with a proposition for a better future for Go. In the previous articles, we [discussed the stronger sides of the language]({% post_url 2022-10-17-what-makes-go-the-best-language %}) and [presented the problematic ones]({% post_url 2022-10-17-we-need-to-talk-about-the-bad-sides-of-go %}). If you missed those, I suggest glancing back as they provide an important context for this one.

#### More in this series
*   [What Makes Go the Best Language]({% post_url 2022-10-17-what-makes-go-the-best-language %})
*   [We Need To Talk About The Bad Sides of Go]({% post_url 2022-10-17-we-need-to-talk-about-the-bad-sides-of-go %})
*   **A Proposition For a Better Future**

If you combine community conventions and the naming problems with async return value problems presented in the previous article, you end up with **hugely popular libraries** shipping code with complex, 100+ line functions, using one-letter undocumented variables, declared at the other side of the package. This is extremely **unreadable and unmaintainable, and surprisingly common**. In addition, as opposed to other modern languages, Go doesn’t provide any kind of runtime value safety. This results in many **value-related runtime issues** which can easily be avoided.

I think as a language and a community, Go tries very much to distinguish itself. A kind of an “_other languages had downsides,_ **_we’ll do everything differently_**” approach. Did other languages make design mistakes? Certainly, and admittedly so — [null pointers](https://en.m.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions) and checked exceptions [in C#](https://www.artima.com/articles/the-trouble-with-checked-exceptions) and [Java](https://radio-weblogs.com/0122027/stories/2003/04/01/JavasCheckedExceptionsWereAMistake.html) are popular examples of such **decisions recognized as mistakes by their designer**.

Did the object-oriented approach get abused over the years? It **most gruesomely did**! In [one of his famous talks](https://commandcenter.blogspot.com/2012/06/less-is-exponentially-more.html), Rob Pike (co-designer of Go) says

> If C++ and Java are about type hierarchies and the taxonomy of types, Go is about composition.

Indeed, composition and inheritance are just different approaches to overcoming code duplication. Composition has its [benefits](https://en.wikipedia.org/wiki/Composition_over_inheritance#Benefits) and [drawbacks](https://en.wikipedia.org/wiki/Composition_over_inheritance#Drawbacks). A decade after Pike’s statement, I hear so many Gophers trashing Java or C++ and their OOP approach over the claim that people abuse inheritance. Let’s make it clear — people will abuse composition as well. Give them time. 😌 Several decades of dominance of composition over inheritance due to the **rise of functional programming** will probably do the trick. In the meantime, I don’t see anything good coming out of it **blindly rejecting good practices** and conventions from other languages. Instead, we should be **carefully examining what we should adopt**. Decades of experience in many other languages must be worth something. And in my opinion, they do a lot.

## A Proposition For a Better Future

I would have loved to see a compile time environment that mostly looks like Go, but allows developers to be a bit more expressive to **gain maintainability and runtime safety**. But at the same time, allow the Go language itself to largely **remain the same** and not evolve into something new, as a lot of us Gophers fear. As Gophers, **why not have two tools in our tool set**? 🔧🪛

![Goat — Extended flavor of the Go programming language, aiming for increased value safety and maintainability](/images/posts/a-proposition-for-a-better-future-for-go/goat-secondary-logo.webp)

Introducing — [**Goat**](https://github.com/goatlang/goat)**.** A new compile-time environment that will produce standard, compatible, and performant Go files that are fully compatible with any other Go project. This means they can import regular Go files but also be safely imported from any other Go file. I want to ignite a thorough discussion around the design and specification of Goat and hear everyone's opinion. Gophers, and non-Gophers. **Come** [**join the discussion**](https://github.com/goatlang/goat/issues), we need your input!

Goat could be implemented in a variety of ways from a fully transpiled programming language to a simple **Go code generation tool**. However, at this point, we intentionally avoid delving into implementation detail and want to focus on spec. **Imagine yourself in the unlimited space of possibilities of such an exciting Go-coding environment, what do you see?** Join the discussion by opening new GitHub issues to propose new syntactic and convention rules, or by joining the discussion in open issues. What would you have changed in Go’s compile time to improve safety and maintainability, and contribute your opinion to the forging spec of Goat.

### What About Fragmentation

Let’s spend a minute going over the solutions presented in the previous article and discussing their consistency and simplicity. Let’s make sure we leave room for Go to remain simple, consistent, and defragmented. We shouldn’t have more than one possible way to express something.

*   Adding visibility modifiers — to keep things defragmented, visibility should be derived **only by visibility modifiers**. In other words, uppercasing and lowercasing symbols will not affect their visibility.
*   Moving built-in symbols to scoped methods and functions — the behavior of built-in functions (like `cap`, `len` and `make`, for example) should only be provided as methods and functions of their respective types. The **only way of acquiring** the length of a slice should be by calling `slice.Len()`. Builtin symbols will not be available.
*   Adding strict nil support — there are several different ways to achieve this at various levels of complexity. One way or the other, the compiler should **prevent assigning nil values to non-nullable variables** and prevent **dereferencing a nullable variable** without initial checks. For those who are not familiar with a similar mechanism, it might take a minute or two to grasp. Either way, no fragmentation should be introduced in the process.
*   Adding enum support — enum support should not overlap with any mechanism currently existing in Go. When you declare a type that **must** hold one of several possible values — enum should be your choice as no other mechanism supports it.
*   Supporting initial values in structs — a mechanism to enforce initial values of struct fields must integrate into an existing mechanism. In other words, we should support **calculating and providing initial values of struct fields within the current struct syntax**. No duplication or fragmentation should be introduced.
*   Adding const assignments — each declaration of a variable should **indicate whether it’s a constant or a variable**. On that note, I’ll add that it requires **removing the support for shorthand variable assignment** (`:=` operator), which currently creates fragmentation in Go syntax.
*   Adding promise support for `go` keyword — it’s important to note that adding support for promises will not introduce a new concurrency model. Rather, it should simply provide built-in support for tracking the execution of fired goroutines and getting their result. This is something that Go doesn’t support today and adding it should be consistent and defragmented.

Disagree with anything? Let us know by submitting a [new Goat issue](https://github.com/goatlang/goat/issues/new).

### What About Other Go-like Languages?

Full disclosure — I’ve learned about the [vlang](https://vlang.io/) project only during the writing of this series. As of writing these words, I’m still learning the specifics of vlang, and I’m always on the look for similar projects. vlang is an amazing and powerful language with, as far as I understand, beautifully designed compile time and run time environments. That being said, there is a **fundamental difference** between `vlang` and similar initiatives, and Goat. While other projects strive to create new and exciting languages, **Goat is all about empowering the existing one**. I believe Go is a fantastic language with a powerful runtime and an amazing ecosystem. I do not wish to lose any of that.

## Summary

After summarizing the upsides and the downsides of Go, we want to focus the efforts on defining the specification of Goat — an extended flavor of Go, aiming for increased value safety and maintainability. This project will allow Go to remain simple and efficient while allowing the community to experiment with an extended flavor. Goat spec should be driven by the community and so it needs the opinion and contribution of any Gopher and non-Gopher out there. [Come join the discussion](https://github.com/goatlang/goat/issues). ❤️
