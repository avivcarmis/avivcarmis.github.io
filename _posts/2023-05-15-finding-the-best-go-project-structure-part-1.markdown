---
date: 2023-05-15 13:12:36
title: Finding The Best Go Project Structure - Part 1
description: Exploring structuring approaches
image: images/posts/finding-the-best-go-project-structure-part-1/go-project-structure-logo.png
categories: [go, architecture]
tags: [go, structure, architecture, clean-architecture, hexagonal-architecture, domain-driven-design, modular-design]
series: go-project-structure
---
TL;DR: This is a story about the journey we've been on at [HUMAN](https://www.humansecurity.com/) Security to find the **best project structure for Go**, what decisions we've made based on our exploration, and the conclusions we've drawn. We've created an open-source [template repository](https://github.com/PerimeterX/go-project-structure) for the final structure, and a branch containing a tiny [example project](https://github.com/PerimeterX/go-project-structure/tree/example) alongside. To use this template, fork the repository or [use it as a template](https://github.com/PerimeterX/go-project-structure/generate). To learn more about it, keep reading.

> This is part one of a two-part series about our work with Go. In this post, we're going to explore the structuring approach, the experience we've had with it, and what made us change our minds.

It's been almost four years since I published [an article]({% post_url 2019-05-19-ok-lets-go-three-approaches-to-structuring-go-code %}) about Go project structure. At the time, I was approaching my one-year mark as a Gopher, and the article reflected my thoughts at the time. I felt there was a lack of structural guidelines and standards from the community which led to confusion and stifled my productivity. Apparently, this resonated well with others. Since then, the article has been read by thousands of people, appeared on [golang-weekly](https://golangweekly.com/issues/266), been [translated into Russian](https://habr.com/ru/company/piter/blog/516186/), and read by several thousands more, appearing on the front page of [related Google searches](https://www.google.com/search?q=structure+go+code).

Over time, I've noticed I rarely meet Gophers who used Go as their native language. Most of us came from other languages and it's easy enough to see based on how we write and structure our Go code. Ex-Java devs tend to write more OOP style or name method receivers `this`. Ex-C, JS, or Python devs also tend to preserve their own habits during the first couple of years. I like to say that Go is a *language of immigrants*.

Go itself is still far away from having a standard for structural guidelines. While a lack of standards can sometimes be nice and approachable, other times it can be counterproductive. A set of best practices, pros and cons, and a predetermined structure can reduce cognitive load and help developers focus on the business value we're trying to achieve.

The original article presented three possible approaches for structuring Go code. If you haven't already, I suggest [going back and reading the full article]({% post_url 2019-05-19-ok-lets-go-three-approaches-to-structuring-go-code %}), but here's a (very) quick recap:

-   **Single Package** - With this approach, you do not create any packages. Instead, you place your entire code base within a single directory.
-   **Coupled Packages** - With this approach, you designate a package (or packages) for defining communication interfaces among all packages. This can dramatically simplify development while allowing the rest of the packages to remain decoupled.
-   **Independent Packages** - With this approach, each package is completely independent of the others. To be able to work together, each package defines interfaces to represent other packages when needed. Then, everything is tied up using dependency injection.

This is broad and unopinionated enough to pass the test of time. Reading it today, I still agree with what it says. While I do, I think it might be a bit **too** unopinionated. Merely an observation of what I've seen out there.

Since the release of the previous article, my company and I have gained tremendous experience with writing and structuring Go code. Over this period, we've had continuous, thorough discussions about the pros and the cons of every decision we've made, and later held retros to review the impact of those decisions. We've had the chance to course-correct, try out different directions, and draw important conclusions. Below, I'll present the structure we use internally today, and work our way through the decisions we've made along the way.

> I should mention that what works for us may not necessarily work for others. I will present the considerations driving us toward each decision, as we mainly write backend applications that handle massive scale, huge data, and real-time decision-making. Other use cases may require other considerations. **Make sure you examine your decisions from the perspective of your use case** and apply them accordingly.

### Structuring Approach

Several years ago we discussed the three different structuring approaches. We wanted to weigh the pros and cons of each approach and decide on the best approach for our needs.

#### Our Initial Goals

Before we began our discussions, we defined two goals we were trying to achieve using our structural design:

1.  **Structural coherence** - each piece of code belongs to a single package in a meaningful way. This provides single responsibility and reduces cognitive load while reading and writing code.
2.  **Separation of business logic and infrastructure**.

Separation of business logic and infrastructure is desired for several reasons:

-   **Testability** - business logic that directly uses a specific infrastructure is less testable.\
    For example, MySQL-specific business logic. Why? First, running even the simplest of unit tests on such pieces of code requires running external dependencies (MySQL server) and seeding them with data.\
    Second, the test results depend heavily on many factors other than pure business logic. Infrastructure code, seed data integrity, integration status, networking, and others. Unit tests are meant to be fast and simple to provide quick feedback. Without a proper separation, we lose this ability. Once we abstract the infrastructure implementation, we can mock infrastructure to directly unit test the pure business logic. This is not to replace or reduce the importance of integration tests. Instead, this is meant to open up the possibility to write simple, reliable unit tests.
-   **Dynamic infrastructure capabilities** - the most basic example of this is switching infrastructure.\
    For instance, if you want to switch from MySQL to MongoDB, it will be harder and much more error-prone if your business logic uses MySQL directly. However, if you abstract that code into separate packages and make sure your business logic is unaware of concrete infrastructure, switching from one technology to another is quite simple.\
    How often do you move your entire infrastructure from one technology to another? Does that alone justify separation? Not necessarily. But this is not the only reason to have a dynamic infrastructure. A more common use case would be to operate several different technologies at the same time. For example, serving requests both on HTTP and gRPC.\
    Or how about intermediate layers? Let's say you use MySQL, but you're planning to add a cache layer. In a separated environment, this is very easy, safe, and intuitive.
-   **Project development capabilities** - if you separate business logic and infrastructure, you have more options while managing projects. For instance, your infrastructure engineer can work on investigating the best database for your use case, while a business logic-focused engineer starts implementing and unit testing the business logic layer. This method allows much more flexibility in project management, and it's notable even when you work alone.

Now that we have goals set up, we started discussing the different approaches.

#### Single Package Approach

```text
.\
|-- database.go\
|-- handler.go\
`-- main.go
```

[Full example](https://github.com/PerimeterX/ok-lets-go/tree/master/1-single-package)

First, we discussed the single package approach. While we acknowledged the ease and simplicity of this approach, we also felt like it can only go so far due to its drawbacks:

-   Lack of private symbols
-   Tight coupling due to no responsibility separation
-   Lack of solid guidelines due to the absence of structure

We considered using the single package approach for small projects and migrating to more complex structures when they grow larger. But we agreed it would only be counterproductive: these migrations are hard to perform and will be too frequent as projects tend to grow.

Additionally, it requires us to be fluent and train for two different structures instead of just one. No, thanks. As an organization dealing with many complex projects, we simply decided to disregard the single-package approach and focus our discussions on the other two.

#### Coupled Packages Approach

```text
.\
|-- database\
|   `-- user.go\
|-- definition\
|   |-- config.go\
|   `-- database.go\
|-- handler\
|   `-- user_permissions_by_id.go\
`-- main.go
```

[Full example](https://github.com/PerimeterX/ok-lets-go/tree/master/2-coupled-packages)

Next, we started discussing the coupled packages approach. We wrote some POC around it and felt like this approach checks our two requirements:

-   Structural coherence
-   Separation of business logic and infrastructure

Although this approach promotes some usage of global variables and negatively impacts reusability, we agreed it is a valid candidate for us.

#### Independent Packages Approach

```text
.\
|-- database\
|   `-- user.go\
|-- handler\
|   `-- user_permissions_by_id.go\
`-- main.go
```

[Full example](https://github.com/PerimeterX/ok-lets-go/tree/master/3-independent-packages)

This approach also meets our requirements. In addition, we also gain **stronger reusability**. Since each package here is fully independent, components can be **safely moved or copied from one project to another**. However, independent packages also mean there's a lot of ceremony. More interfaces must be written and maintained to support package decoupling, and a more extensive reference passing and dependency injection must be performed.

It's an interesting balance. Both sides seem like they can hurt development velocity, but it's hard to estimate the impact of each. After discussing the pros and cons of each side we came to a conclusion. How often do you need to move components from one project to another? It seemed then like a luxury worth sacrificing to be able to write simple code quickly. We agreed on the coupled packages approach. Spoiler alert: we later decided we were wrong.

We went on to write coupled packages-based projects for several years. Throughout this time, we've held continuous discussions about structuring decisions, however, we were patient about opening up an official retrospect about the whole structuring approach. By now, though, we've gained enough experience working with the coupled packages approach and were in a position to analyze it.

### Retrospective Analysis

One thing was clear: coupled packages structuring is a solid approach. It provides full separation of business logic and infrastructure, testability, and ease of use. This was powerful enough to have us going for several years. However, it did have important shortcomings.

#### Reusability Issues

Initially, we defined reusability as the ability to share and move code **between different projects**. While we still agreed this was a luxury, we missed a far more important feature of reusability: the ability to share and move code **within the project itself**.

In coupled packages projects, sharing and reusing code is simply unsafe due to the usage of global variables. Let's see an example. Suppose we have an HTTP server exposing some secret value for users who provide a valid token. We want the HTTP port and the token to be provided via environment variables. For this, we create a config package:

```go
// file: config/env.go

var (
	Port   = os.Getenv("PORT")
	Token  = os.Getenv("TOKEN")
	Secret = os.Getenv("SECRET")
)
```

And an HTTP package:

```go
// file: http/secret_server.go

type SecretServer struct {
	server *http.Server
}

func NewSecretServer() *SecretServer {
	s := &SecretServer{}
	mux := http.NewServeMux()
	mux.HandleFunc("/get_secret", s.getSecret)
	s.server = &http.Server{Addr: config.Port, Handler: mux}
	s.server.ListenAndServe()
	return s
}

func (s *SecretServer) getSecret(writer http.ResponseWriter, request *http.Request) {
	if isValidToken(request, config.Token) {
		tellSecret(writer, config.Secret)
	}
}
```

Simple enough. Now, what happens if I want to add an additional HTTP server, serving on a separate port, exposing a different secret with a different token?

As you can see, since the code for `SecretServer` is generic, I can reuse this struct to support the new server. However, it contains hard-coded references to configuration values that tie this struct to a specific port, secret, and token. The solution seems quite simple. Parameterize everything via the constructor, like this:

```go
// file: http/secret_server.go

type SecretServer struct {
	server *http.Server
	token  string
	secret string
}

func NewSecretServer(port string, token string, secret string) *SecretServer {
	s := &SecretServer{token: token, secret: secret}
	mux := http.NewServeMux()
	mux.HandleFunc("/get_secret", s.getSecret)
	s.server = &http.Server{Addr: port, Handler: mux}
	s.server.ListenAndServe()
	return s
}

func (s *SecretServer) getSecret(writer http.ResponseWriter, request *http.Request) {
	if isValidToken(request, s.token) {
		tellSecret(writer, config.Secret)
	}
}
```

If you're experienced enough, you might see the bug here. I mistakenly updated everything **except for the secret itself**. This small issue can easily create *catastrophic effects* on a business. Angry customers, blocked users, unresponsive servers, etc.

What can we do? The usage of global variables is what's causing this issue. Blocking the ability of the HTTP server to directly pull these config values would have forced you to have everything parameterized in the first place. Remember the tradeoff? A bit more ceremony when writing and maintaining code? The result is not merely "reusability". It's about safety and assurance, The ability to easily reason about code, and reuse it without unnecessary risk.

#### Undeclared Dependencies

To demonstrate the issue with undeclared dependencies, let's say we have an application communicating with two databases, one for users and another for user tasks. We first create a DB package, managing the dependency injection and the references for the database instances:

```go
// file: db/instances.go

var (
	UserDB      UserDBInterface
	UserTasksDB UserTasksDBInterface
)

type UserDBInterface interface {
	UserByID(id string) *User
}

type UserTasksDBInterface interface {
	TasksByUserID(id string) []*Task
}
```

Then we have the business logic layer:

```go
// file: core/users.go

func UserAndTasksByID(userID string) (*db.User, []*db.Task) {
	user := db.UserDB.UserByID(userID)
	tasks := db.UserTasksDB.TasksByUserID(userID)
	return user, tasks
}
```

In this example, `core.UserAndTasksByID` depends on both `db.UserDB` and `db.UserTasksDB`. It assumes both are initialized and ready to operate. You can document the dependencies in various ways, but documentation tends to grow outdated or unread. More importantly, they don't provide any ability for compile time enforcement. If those two packages can't communicate directly, the core layer must receive its dependencies as arguments. This pattern provides documentation (or better yet, a contract). But more importantly, **compiler enforcement over dependency usage.**

#### Changing Our Approach

Overall, we had a positive experience working with the coupled packages approach. Changing direction and switching to independent packages means writing and maintaining more code. Explicitly declaring dependencies and passing them around. Given the pain points described above, we were more than willing to pay the price.

Over the last couple of years, we've been writing and migrating to the independent packages approach, and the experience was strongly positive. In addition to gaining safe and simple reusability and visibility over dependency usage, It forced us to better understand the structure of our projects, the inner dependencies, and hierarchies, and it exposed many mistakes and unused resources the global variables approach made us overlook.

### What's Next?

In the [second part of our series]({% post_url 2023-05-15-finding-the-best-go-project-structure-part-2 %}), we'll discuss architectural design, package structure, and directory structure.

### Connecting All The Dots

We're not done describing the entire structure and the decision-making process. However, if you're anxious to see the whole picture, we've created an example repository implementing our structural rules and guidelines. The [example branch](https://github.com/PerimeterX/go-project-structure/tree/example) contains a simple TODO application to manage tasks. The [main branch](https://github.com/PerimeterX/go-project-structure) contains an empty skeleton for creating new repositories. To use the structure, fork the repo, or [use it as a GitHub template](https://github.com/PerimeterX/go-project-structure/generate). Have any questions? Disagreements? We would like to know and discuss them. Submit a GitHub [issue](https://github.com/PerimeterX/go-project-structure/issues), or comment on social media ❤️

[![Go Project Structure Repo](/images/posts/finding-the-best-go-project-structure-part-2/go-project-structure-preview.png)](https://github.com/PerimeterX/go-project-structure)
