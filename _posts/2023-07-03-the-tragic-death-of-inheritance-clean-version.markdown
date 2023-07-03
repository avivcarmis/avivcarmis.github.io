---
date: 2023-07-03 11:12:44
title: The Tragic Death of Inheritance (Clean Version)
description: This is a story about how I gradually fell in love with composition
image: images/posts/the-tragic-death-of-inheritance/the-tragic-death-of-inheritance.jpg
categories: [inheritance, composition, oop]
tags: [oop, inheritance, composition, maintainability]
---
### The Tragic Death of Inheritance (Clean Version)

**TL;DR** - this is a story about how I felt forced to give up inheritance and object-oriented programming, and still missed it for a very long time. And why now, half a decade later, I believe that inheritance is an inferior choice in most cases.

*Note: this is a clean version of the post. If you're ok with explicit language, I suggest reading the [original version]({% post_url 2023-06-14-the-tragic-death-of-inheritance %}).*

### An Embarrassing Change of Hearts

About five years ago I started writing Go. Earlier in my career, I had a diverse background including Java, JavaScript, Python, C#, PHP, and others. Each of those contains some level of object-oriented support, either by built-in constructs or by lack of typing. Go, on the other hand, doesn't. There's no inheritance, no abstract classes, none of that.

After the first four years of extensive Go coding, and even while leading the way with developer experience for my company, I'd still argue for inheritance. I'd initially say inheritance is better than composition, then I went through the phase of admitting they're both valid means to an end, but that I, personally, prefer inheritance. And finally, after having written Go for almost five years, it hit me. Composition is better than inheritance. 🫳🎤 *Dramatic pause*...There. I said it. So why do I think so now? Why didn't I earlier? Let's try to break it down.

Changing your mind is never easy, even more so after publicly voicing your opinion in various posts. I want to back it up with some concrete facts about the two programming paradigms: **inheritance** and **composition**. We'll start with some background, then head down to the phases of my realization, explaining the advantages inheritance does have over composition, and finally, explain the benefits of composition, making it my new favorite programming paradigm. 🤦‍♀️ That felt weird even writing it down.

### Background

Composition vs. inheritance is not an easy topic to cover. I don't think there's a clean cut where composition ends and inheritance begins. There's a lot of grey area in between. I tried pulling the definition from Wikipedia's [*Composition Over Inheritance*](https://en.wikipedia.org/wiki/Composition_over_inheritance) and ended up with some explanation about polymorphic something and a link to a book. Nah...that's not going to be good enough. I need to be able to express it in simple words. Even if it won't be completely accurate it would still give me better clarity.

Let's start with *why?* Why do we need either of them? So, mainly to **avoid code duplication**. Code duplication creates error-prone code bases. Composition and inheritance are two different tools to eliminate it. So what exactly is the difference between the two? While **inheritance is all about hierarchies**, **composition is about composing building blocks**. Like Jigsaw Puzzle. 🧩

Allowing myself to go the least accurate and most concrete with my definition: **inheritance is about "is-a" relationship** between components, and **composition is about "has-a" relationships**. Inheritance would claim that a **dog is an animal**, while composition would probably argue that a **dog has some animal characteristics**.

Let's take an example problem to help us through this entire discussion. We have three types of animals that share some behaviors but differ in others:

```ts
interface Animal {
    doStuff()
}

class Dog implements Animal {
    doStuff() {
        eat()
        breathe()
        println("woof")
        sit()
        sleep()
    }
}

class Cat implements Animal {
    doStuff() {
        eat()
        breathe()
        println("meow")
        sit()
        sleep()
    }
}

class Pig implements Animal {
    doStuff() {
        eat()
        breathe()
        println("oink")
        sit()
        sleep()
    }
}
```

In this example, `Dog`, `Cat`, and `Pig`, contain a very apparent code duplication.

How would inheritance solve this problem:

```ts
interface Animal {
    doStuff()
}

abstract class AbstractAnimal implements Animal {
    abstract speak()

    doStuff() {
        eat()
        breathe()
        this.speak()
        sit()
        sleep()
    }
}

class Dog extends AbstractAnimal {
    speak() {
        println("woof")
    }
}

class Cat extends AbstractAnimal {
    speak() {
        println("meow")
    }
}

class Pig extends AbstractAnimal {
    speak() {
        println("oink")
    }
}
```

Obviously, inheritance has many different solutions for this problem. However, if this looks familiar, it's because this is a rather typical inheritance approach.

How would composition solve this problem:

```ts
interface Animal {
    doStuff()
}

class Dog implements Animal {
    doStuff() {
        consumeEnergy()
        println("woof")
        rest()
    }
}

class Cat implements Animal {
    doStuff() {
        consumeEnergy()
        println("meow")
        rest()
    }
}

class Pig implements Animal {
    doStuff() {
        consumeEnergy()
        println("oink")
        rest()
    }
}

function consumeEnergy() {
    eat()
    breathe()
}

function rest() {
    sit()
    sleep()
}
```

Since composition is not as structured as inheritance, it has a wider variety of approaches. However, this is probably the simplest composition solution. A different composition solution would be a bit more complex but eliminate more code duplication. It will be leveraging...you guessed it, function composition:

```ts
interface Animal {
    doStuff()
}

class Dog implements Animal {
    doStuff() {
        genericDoStuff(function() {
            println("woof")
        })
    }
}

class Cat implements Animal {
    doStuff() {
        genericDoStuff(function() {
            println("meow")
        })
    }
}

class Pig implements Animal {
    doStuff() {
        genericDoStuff(function() {
            println("oink")
        })
    }
}

function genericDoStuff(speak) {
    eat()
    breathe()
    speak()
    sit()
    sleep()
}
```

Since inheritance and composition are two completely different programming paradigms, they create different code structures. When you get used to seeing one of them, the other seems wired. Shifting between them means **changing your mindset.** And trying to mix the two will probably lead to something you aren't pleased with.

### Phase 1 - Denial

As someone with a fair share of experience with OOP, composition initially seemed to me like code duplication. If you look at the examples above, inheritance is obviously cleaner. Each component contains the bare minimum code to specify its deviation from the default behavior. No code duplication at all. The composition examples, on the other hand, contain some portion of code duplication. 🥴 This is not a pretty place to start with for composition.

Furthermore, inheritance requires some level of syntactical support. The language must allow inheritance to exist, otherwise, you simply can't write it. For composition, you might need support for higher-order functions. But even without them, you can still implement composition. Just make functions call other functions. It's funny that we've even bothered giving it a name. It's just basic programming, right? 😐

And that's what makes it even harder. As someone who's used to seeing syntactical structures completely eliminating code duplication, to suddenly see *just basic programming,* which leaves at least some level of duplication in the source code, and I'm supposed to...what, see it as a step forward? I mean...come on 🤦‍♀️ I think we can all agree that inheritance has been abused for way too long, but if you're going to dismiss it, at least come up with a better solution than that! 🫵🤨

### The Problems With Inheritance

But inheritance does have its downsides. Before we dive into them, let me just start by saying this: readability is more important than writability. This is another typical opinion in the Go community, but I think it goes beyond one language or another. Allow me to repeat something I've already written in a different post:

> *Code should be easier to read than it is to write. How can you argue with that logic? Every professional developer knows that* ***you write a line of code once and then maintain it for 5 years***.

I couldn't stress enough how strongly I feel about this.

Now let's talk about readability issues with inheritance. By nature, **inheritance is a bottom-up approach**. Take another look at the inheritance code from above:

```ts

class Dog extends AbstractAnimal implements Animal {
    speak() {
        println("woof")
    }
}
```

If you wanted to figure out the behavior of `Dog.doStuff()`, you'd first have to go up the hierarchy and understand the behavior of `AbstractAnimal` to figure out how exactly it interacts with its abstract methods. Then you have to start traveling up and down the tree to figure out which methods are overridden and at which level. This is extremely demoralizing when you're not the one who wrote the code, and the deeper and more complex the hierarchy is, the harder it gets.

Most of us have bad memories of over-complex hierarchies we regret seeing, but this is probably due to the dominance of OOP in recent decades. I can easily abuse either inheritance **or composition** to the point that they're unreadable. I do not wish to base the entire discussion on extreme cases, instead, just acknowledge one basic fact. Composition **manages control flow** by the function called on **the actual call site**, while inheritance may do it by a method of an ancestor up the hierarchy tree. This makes the entire flow harder to grasp. Instead of seeing the whole picture and then diving into details, you first see some details, and now you have to traverse the hierarchy to understand the whole picture.

If you quickly want to get a grip on what's going on in `Dog.doStuff()`, or better yet if you want to reason about the behavior of the code, or identify concurrency issues or bugs, isn't this clearer to immediately get a sense of:

```ts
class Dog implements Animal {
    doStuff() {
        consumeEnergy()
        println("woof")
        rest()
    }
}
```

Another problem with inheritance is: inherited code is less dynamic by nature, and future changes may require much more effort.

What do I mean? Let's say we want to add some behavior to dogs. Right after they eat, and just before they breathe, we want them to get a treat. In composition, this is a simple task: go to `Dog` class, add `getTreat` call in between `eat` and `breathe`. Done. 💪

With inheritance, we probably go somewhere along this:

```ts
abstract class AbstractAnimal {
    abstract speak()

    doStuffInBetweenEatingAndBreathing() {}

    doStuff() {
        eat()
        doStuffInBetweenEatingAndBreathing()
        breathe()
        this.speak()
        sit()
        sleep()
    }
}
```

Do not underestimate the implications here. This is not always trivial to figure out the best implementation for such a requirement. Where would you break the ancestor behavior to allow for this change? Are you able to implement it only by changing `Dog` and `Animal`, leaving the rest of the classes changeless? What would you call it? Is `doStuffInBetweenEatingAndBreathing` a good name? Probably not. How about `getTreat`, is it better? Isn't it too specific to dogs? In real-world scenarios, base classes like this have so many non-trivial breaking points to support such specific cases the whole thing becomes completely unmaintainable. What about performance implications? Do we actually want cats and pigs calling an empty function just to allow dogs to get their treats? Even considering possible compiler optimizations it all just seems too much. Even for a GOOD BOI 🐶🐶

But why does it all happen? It happens because hierarchies are focused on similarities. Once you designate some area in the code as shared between several components, you assume they all act alike. Any future deviation inside this area makes you force this new behavior on the entire sub-tree. This is what inheritance does, it makes you identify taxonomies and inherited structures everywhere. Many times this is accurate enough and the result is very clean and elegant. Other times you end up with a grotesque creature staring you in the face asking "Mommy...why you make me like this mommy?" 🥴

### Phase 2 - Internal Debate

During my years either as a passionate hobbyist developer, an academic student, a professional engineer, and an architect, I learned and even taught that code duplication is bad. Back to Wikipedia, this time pulling a very clear and simple definition from [*Duplicate Code*](https://en.wikipedia.org/wiki/Duplicate_code#Costs_and_benefits):

> "[Duplicate code] is simply longer, and if it needs updating, there is a danger that one copy of the code will be updated without further checking for the presence of other instances of the same code...When code with a software vulnerability is copied, the vulnerability may continue to exist in the copied code if the developer is not aware of such copies...Refactoring duplicate code can improve many software metrics, such as lines of code, cyclomatic complexity, and coupling. This may lead to shorter compilation times, lower cognitive load, less human error, and fewer forgotten or overlooked pieces of code."

There is no doubt all of this is true and highly important, but there is also a distinction to be made: not all code duplications are the same. **Some impose higher risks than others**. For example, take a look at this code:

```ts
function insertA(db) {
    connection = db.getConnection()
    connection.validateTableExist("table_name")
    connection.table("table_name").insert("a")
}

function insertB(db) {
    connection = db.getConnection()
    connection.validateTableExist("table_name")
    connection.table("table_name").insert("b")
}

function insertC(db) {
    connection = db.getConnection()
    connection.validateTableExist("table_name")
    connection.table("table_name").insert("c")
}
```

Does it hurt your eyes? It should. Most potential changes we make to this code snippet pose a high risk of us overlooking duplications. How do you feel about this:

```ts
function insertA(db) {
    getTable(db).insert("a")
}

function insertB(db) {
    getTable(db).insert("b")
}

function insertC(db) {
    getTable(db).insert("c")
}

function getTable(db) {
    connection = db.getConnection()
    connection.validateTableExist("table_name")
    return connection.table("table_name")
}
```

Is there a potential risk? There's less, but there still is. What if I want all insert functions to close the DB connection before they return? I can easily refactor it, but I still have to avoid overlooking duplications while I do. The important question here is this: **is the second version better than the first?** It's not perfect, but you must agree it's at least slightly better. **There are simply fewer duplications.**

As with other software challenges, it's a matter of tradeoff. Taxonomies might be less readable but contain less code. In cases where a real-world problem can be well-modeled into a taxonomy, inheritance makes a good case. While the end result with composition might not be as elegant or clean, it makes sense to have some carefully chosen code duplications to avoid bad taxonomies and the issues that go along with them.

### Phase 3 - Acceptance

Being the selfish engineer that I am, the realization finally hit me not while writing my own code, but while reviewing peers. When I write code, I want it to be perfect. I want no code duplications, I want it to be beautiful. The thing that's hard to appreciate and fully comprehend, is the fact that without the heavy context and knowledge you hold in your brain while writing code, it always looks compilated.

Inheriting (🥁) someone else's code, 6 months after they've left the company, is never fun. You go over it while scratching your head, asking "But why? Why would they do that?". Sounds familiar? To me, it happens even if I admire that person. Know this about me - while I read your code, I do not admire you. I actually think you made a terrible mistake. Here. And here. And also there. Why do you overcomplicate everything?... God knows this is what everyone else has on *their minds* when they read *my code*. 🤦‍♀

What's the solution? The code you write has to be **dead simple**. I'm fine with some code duplication as long as it's well intended, and it helped you write a straightforward, top-down code that I can read and immediately understand your intentions, the outcome, the concurrency complications, and remember how much I admire you. ❤️

In some ways, I expect experienced engineers' code to resemble that of someone new to programming. The more experienced you are the less tempted you should feel to take the fancy solution to a problem.

### Advantages of Inheritance

From what we've shown so far, it might seem like the only advantage of inheritance is a stronger elimination of code duplication, with the addition of a more elegant, cleaner code. This is not the whole picture. Inheritance has some more tricks up its sleeve.

We talked about how inheritance-based code is less dynamic when changes are introduced. This means future requirements might not completely fit the current modeling of the problem, which either leads to remodeling of the code or forcing the current model to support the new requirements.

While this is true for modeling new requirements, actually coding them might be safer with inheritance. Both the complete lack of code duplication and the compiler guarantees over abstract classes provide safer environments than the ones provided by composition.

For instance, if we want our animal's `doStuff` method to run its logic in an infinite loop, inheritance change will be super safe due to a complete lack of code duplication:

```ts
abstract class AbstractAnimal {
    abstract speak()

    doStuff() {
        while (true) {
            eat()
            breathe()
            this.speak()
            sit()
            sleep()
        }
    }
}
```

With composition, we might change `Dog` and `Cat` but forget `Pig`. This is the biggest fear with code duplication, and you have to be very careful while making such changes.

Another example: what happens if we want our animals to make different sounds when they wake up? Some custom logic to perform once returning from `sleep` function. Inheritance provides a safer experience due to compile time guarantees:

```ts
abstract class AbstractAnimal {
    abstract speak()
    abstract wakeUp()

    doStuff() {
        eat()
        breathe()
        this.speak()
        sit()
        sleep()
        this.wakeUp()
    }
}
```

At this point, the code simply doesn't compile until you've updated all child classes. This isn't the case in most composition implementations, which may easily lead to neglecting to change all places.

### Final Thoughts

With classic object-oriented languages like Java and C# showing signs of passing their peak period, and languages like Rust and Go showing a consistent increase in interest ([source](https://madnight.github.io/githut/#/issues/2022/4)) while being quite vocal against inheritance, it seems like the programming community has had it with inheritance.

While I strongly agree that object-oriented programming and specifically inheritance got abused over the years, I mostly felt like denying inheritance altogether was somewhat childish and no more than a simple trend.

I still believe inheritance can be a good solution when modeling a real taxonomy-based problem. In all other cases, we should probably avoid it, even though the resulting code may seem uglier. The truth is I was blinded by a dogmatic belief that any code duplication is nasty. Once I let it go, I started seeing something else. In many cases, inheritance makes our code less readable and less maintainable in the name of keeping it clean. While I enjoy modeling and writing inheritance, understanding and maintaining it could get worse and worse.

As professional coders, delivering actual business value, we have to ask ourselves what's more important: writing beautiful and elegant code, or writing code that others will be able to easily understand and maintain.