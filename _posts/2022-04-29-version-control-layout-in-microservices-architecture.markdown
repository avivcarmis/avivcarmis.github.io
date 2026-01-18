---
date: 2022-04-29 13:12:27
title: Version Control Layout in Microservices Architecture
description: The pros and cons of version control layout alternatives
image: images/posts/version-control-layout-in-microservices-architecture/version-control-layout-in-microservices-architecture.webp
categories: [microservices, architecture, developer-experience, version-control]
tags: [microservices, architecture, developer-experience, monorepo, multirepo]
series: developer-experience-in-microservices
---
TL;DR - this is a discussion about the pros and cons of version control layout alternatives. There is a ton of reading material regarding the pros and cons of monorepo vs. multirepo. In this article, I intend to argue that the **decision should be derived mainly from production architecture**.

<div class="series-nav">
  <div class="series-nav__title">Developer Experience in Microservices</div>
  <div class="series-nav__subtitle">Article 2 of 4</div>
  <div class="series-nav__list">
    <a href="/designing-developer-experience-in-microservices-architecture" class="series-nav__item">1. Introduction</a>
    <span class="series-nav__item--current">2. Version Control Layout ← You are here</span>
    <a href="/code-sharing-in-microservices-architecture" class="series-nav__item">3. Code Sharing</a>
    <a href="/testing-strategies-in-microservices-architecture" class="series-nav__item">4. Testing Strategies</a>
  </div>
</div>

## Version Control Layout

Version control layout in a microservices architecture is a basic strategic decision that will later dictate the options available for the next strategic decisions. Version control layouts are usually depicted as the battle between monorepos 💣 and multirepos 💥, each has a long list of pros and cons which we'll shortly discuss. However, I'd like to argue that the strongest consideration should be  - **correctly capturing production architecture**. This will later allow us for stronger validation and assurances before going live. 😇 🤝

## Monorepo

A monorepo is a single version control repository containing several applications, libraries, and components. Typically, these components will compose a system. A simple example:

```text
my-projects
├── my-monorepo-repository <this is a repository>
│   ├── my-backend-application
│   ├── my-frontend-application
│   └── my-shared-code
```

## Multirepo

The multirepo approach advocates a dedicated version control repository for each individual component. The same example, now with multirepo:

```text
my-projects
├── my-backend-application-repository <this is a repository>
├── my-frontend-application-repository <this is a repository>
└── my-shared-code-repository <this is a repository>
```

## Hybrid Solutions

Hybrid approaches advocate creating combinations of the above. Tools such as [meta](https://github.com/mateodelnorte/meta) and [Git-X-Modules](https://gitmodules.com/) allow us to create dedicated repositories for each component and also create a parent repository to reference all sub repositories and group them together. This provides a wrapper to allow synchronizing operations across more than one repo, it can help reduce the time and complexity of such operations. 💪
Another hybrid approach example would be using `git subtree split` to split a monorepo to several dedicated component repositories and manage them apart while allowing for explicit merging and syncing. 🤜 🤛

## Consistent Version Guarantee

In the series intro, we've defined [consistent version guarantee]({% post_url 2022-04-21-designing-developer-experience-in-microservices-architecture %}#consistent-version-guarantee) between components A and B as:

> Each version of component A is guaranteed to interact exclusively with a certain version of component B.

(If you haven't already, [quickly view the intro section]({% post_url 2022-04-21-designing-developer-experience-in-microservices-architecture %}#consistent-version-guarantee) to better understand guaranteed version interaction and to view some examples).

## Choosing The Right Layout

So what's the decision-making process? I believe that matching the version control layout to production architecture will later have the strongest ROI. It will empower our development teams with highly trustable automated testing processes and increase production reliability while maintaining maximum simplicity. 🙌 😌

In a **consistent version environment**  - monorepos provide everything we need in our automated testing while keeping everything as simple and smooth as possible. 👏

For **inconsistent version environments**  - multirepos are required to automatically validate backward compatibility and provide production-oriented testing to increase reliability. 🍻

More on that in the last piece of this series  - [Testing Strategies in Microservices Architecture]({% post_url 2022-05-08-testing-strategies-in-microservices-architecture %}).

## Other Considerations

So what about other considerations? Let's quickly cover them.

#### Searchability 🔍

Monorepos take a clear win here. Cross component search and navigation are much easier, quickly jumping to definitions and documentation is very simple and straightforward. In multirepo environments, this is also possible, even without cloning all the repos. GitHub, for example, provides such tooling - organization-level search and jump-to-definitions. However, monorepos are definitely built for this. 🏃‍♀

#### Coding Standards 👨‍💻

If consistent coding standards are important - monorepo can get a slight advantage here. Obviously, automated tools are required to enforce policies and standards and this will work the same for monorepos and multirepos. However, teams who share the same repo are more likely to retain shared culture at scale. 👥

#### Version History & Information 📖

Multirepos take a strong win here. Each component has its own dedicated release and version control history. There might be tools to empower monorepos for such needs, however, I'm not familiar with any. If you are, please let me know in the comments! 🖊 ❤️

#### Parallelism ⚡️🏃🏻

Multirepos take a slight win here too due to a bit less chance for conflicts, concurrent merges, and a high-rate need for pulling.

#### Access Control 🚫

Multirepos have granular control over permissions and access control per component. Again, monorepos might support it too, depending on the version control platform and its features.

#### Cross Component Refactoring 🚧

This point is really a double-edged sword, monorepos definitely provide us faster abilities to refactor across multiple components. However, in some cases deploying such changes can get dangerous. More on that in the [final piece of this series discussing testing strategies]({% post_url 2022-05-08-testing-strategies-in-microservices-architecture %}).

#### Cloning Time ⌚️

On one hand, cloning can take many hours in extreme cases of monorepos. On the other hand, it's a one-time process and then you never have to deal with cloning again. I'd say it's a matter of taste.

#### Dependencies & Code Sharing 🔗

Strong win for monorepos. Dependencies and code sharing become as simple and straightforward as possible. Since this is a very important consideration, a wide variety of tooling and techniques open all possibilities to multirepos. However, since there is no truly simple solution for multirepos, monorepos win here mainly on the ability to remain clear and simple.
This is a very important point on its own, and I want to clearly discuss the approaches available for code sharing in multirepos, keep reading into the next article, we will do exactly that. 😁 🤗

## Conclusion

Generally speaking, all considerations provide tooling to support both if needed. The strongest point **of monorepos is simplicity**. In the third article in this series we will discuss testing strategies and argue that in inconsistent version environments, it might be worth **sacrificing simplicity to gain production reliability**, which gives the edge to multirepos. 💪

Next, we'll be discussing [code sharing in part 2/3 of the series]({% post_url 2022-05-03-code-sharing-in-microservices-architecture %}).

## What Have I Missed?

Have I missed anything? **Please share your thoughts** and let me know. ❤️

Special thanks to my superstar colleague [Vlad Mayakov](https://www.linkedin.com/in/mayakov-vlad/) for helping in making sense and putting it all together.