---
medium: https://medium.com/perimeterx/the-case-for-git-submodules-d3dad05187fb
date: 2022-05-03 10:03:54
title: Code Sharing in Microservices Architecture
description: Leveraging various tools to empower code sharing in a microservices architecture
image: images/posts/code-sharing-in-microservices-architecture/code-sharing-in-microservices-architecture.webp
categories: [microservices, architecture, developer-experience]
tags: [microservices, architecture, developer-experience, monorepo, package-manager, git-submodules]
series: developer-experience-in-microservices
---
TL;DR - This is a story of leveraging various tools to **empower microservices architecture**, how we use it at [HUMAN](https://humansecurity.com/), and when to use and not use it. 💪

\* This is part 2 of a 3 piece article series discussing key decisions in designing developer experience and development cycle for teams in a microservices architecture. 💡🚀
- [Designing Developer Experience in Microservices Architecture (intro)]({% post_url 2022-04-21-designing-developer-experience-in-microservices-architecture %})
- [Version Control Layout in Microservices Architecture (pt. I)]({% post_url 2022-04-29-version-control-layout-in-microservices-architecture %})
- **Code Sharing in Microservices Architecture (pt. II)**
- [Testing Strategies in Microservices Architecture (pt. III)]({% post_url 2022-05-08-testing-strategies-in-microservices-architecture %})

## Code Sharing

Let's start with some motivation - why do we *need* to share code? 🤔
Well, there can be several reasons for this. The most common cases will include sharing internal libraries, abstractions, common code, models, or entities between several different applications.

So what are the options to tackle this? Here's what I have:

-   Using a monorepo.
-   Avoid sharing code completely.
-   Manual sharing - manually copying and pasting between repositories.
-   Using the package manager.
-   Using git submodules.

Let's go over each option and spend we minute weighing it in. 🏋️‍♀️

## Monorepo

We've discussed version control layout and its implications in the [previous article]({% post_url 2022-04-29-version-control-layout-in-microservices-architecture %}). As discussed, one of the main advantages of monorepos is the ease of code sharing. 😁
If you do work in a multirepo layout, or at least you do want to share code between several repositories, keep reading. 📖 👀

## Avoid Sharing Code

If that's feasible it's probably the best option. If your use case enables you to reasonably split the code between services, this is going to save you many hours of suffering! 😓 I strongly suggest this option if it makes sense.

## Manual Sharing

I don't think we have to deep dive into why code duplication is not recommended, however, if your use case requires sharing only a tiny bit of code that you'll never need to change - this might be a considerably good option. 😮

## Package Manager

Probably the most popular option out there. When we need to share code between several applications we tend to use our package managers (`npm`, `pip`, `maven`, `gradle`, `cargo`, `go mod`, etc, etc, etc). 🏗
This option has several strong advantages. For starters, we already use the package manager for external dependencies, it doesn't require new tooling or additional knowledge. Second, since this approach is very popular, there's a ton of reading material and tools to support it (a very popular one being JFroog's Artifactory, for example).

However, it's not always the best approach. First, setting it up is not very easy. Package managers do not support private dependencies out of the box. Configuring and maintaining them can be tough, expensive, or some combination of both. 😓 🤑
If you already use such a setup then you're probably familiar with everything I have to mention, feel free to skip to the next paragraph. If not, I'll just briefly mention that most package managers require a private registry to host your private repositories (this either costs a lot of money, or it requires a lot of knowledge and a bit of money to create and maintain in scale). Then you'll need to set up your package manager to be familiar with your private registries and provide it permissions to fetch dependencies. You'll need to do it both on the local machine of every relevant employee in the organization and in CI/CD or any other build processes you have set up. 😵

As annoying as it is, it's completely doable, very popular, and mostly a one-time effort. However, I'll argue that in a lot of cases this approach has yet another drawback that dramatically hurts development cycle and developer experience in general. Let's consider the following use case:

![2 service repositories using the same shared repository as a dependency](/images/posts/code-sharing-in-microservices-architecture/code-sharing-in-microservices-architecture.webp)

Let's assume we currently work on our *SweetApplication* and we find an issue in our *AwesomeUtil*. We cannot directly edit *AwesomeUtil* under *SweetApplication* since it's managed by the package manager (effectively, the files are either read-only or may be overridden at any time). You'd have to open a local clone of *AwesomeUtil*, branch out of whatever target you work with, and fix the issue. Before you push and merge your fix you'll need to test it via *SweetApplication* to make sure this solves **the original problem**. 😮
Most package managers support some kind of `replace` statement which will make *SweetApplication* use the local clone of *AwesomeUtil*. Now you can run *SweetApplication* and actually test your fix. Note that code navigation and debugging within *AwesomeUtil *are not always fully available, depending on the package manager, the IDE, and the debugger.
When finally everything works as expected you can go ahead and publish the fix to *AwesomeUtil* (this usually involves push, review, merge and finally release/publish to your private registry). Now you have to **remember** (sure, remembering was never an issue for me 🤦‍♀) to remove the `replace` statement you've added before, update the version of *AwesomeUtil* via the package manager and only then, push your code. 😅

This whole thing is totally feasible, the main question is **what is your use case?** If your shared code is a private library, completely disconnected from business logic needs, matured over a while, and stable, it's likely not to require frequent changes. For such a use case the pros can outweigh the cons. 👌

## Git Submodules

Git submodules offer the same control over version management, but they provide a much faster development cycle. With *AwesomeUtil* as a git submodule, you can edit its files directly from *SweetApplication* **exactly how you imagine it**. You have both repositories locally, you can navigate the code, switch branches, perform changes, run, test, and debug. Finally, when you know everything works well - push your branches in both repos and you're done. 😱🤝

So why not use it in all use cases? For starters it doesn't replace the package manager, so now you have to use the package manager for public dependencies and git submodule for private ones. Additionally, the mechanism is not always very easy to figure out and people usually don't possess the required knowledge. When you sign a new Python engineer, she's probably familiar with `git` and `pip`, but not with the **mechanism behind git submodules**, and without the proper training, it can get confusing. 📚 🏋️‍♀️

Some other considerations? On the good side, git handles private repos seamlessly, since your parent repo is already private (it must be if it depends on a private repo, right?) and git has permissions to clone it. ✅
On the bad side, what happens when the parent repo and child repo both have a common submodule but with a different version? 😵 Package managers are built to handle these cases, git submodules aren't as much. 👎

Let's (very briefly) go over the key points of git submodule usage.

-   **Adding** *AwesomeUtil* to *SweetApplication* as a git submodule is done using the following command:
    `git submodule add https://path.to/repository`
-   When **cloning** *SweetApplication*, you need to make the package manager fetch dependencies. Similarly, you need to make git initialize submodules. It's done using the following command:
    `git submodule update --init --recursive
    `this will make sure the local copy of *AwesomeUtil* is initialized.
-   When you want to make *SweetApplication* use a **different version** of *AwesomeUtil*, you simply checkout whatever revision you want of *AwesomeUtil*. That means `cd` into *AwesomeUtil* subdirectory and `git checkout <revision>`, where revision can be any branch name, tag, or commit ID you desire. Then `cd` back up to *SweetApplication* and you'll see you have an uncommitted change, you can commit and push this change on its own or alongside other changes, just like you would have with changes to `package.json`, `requirements.txt`, `pox.xml` or whatever package manager you use. This is how versioning is performed in git submodules. Simple and safe. 💪
-   When **switching a branch** in *SweetApplication*, it might use some different versions of some dependencies. To sync the changes, you need to make the package manager update dependencies. Similarly, the revision of *AwesomeUtil* may have changed, to sync it you need to make git update submodules. It's done using the following command
    `git submodule update`

The last point can be a bit tricky, I haven't come across any IDE integration to automatically notify when I need to update git submodules (like it does with some package managers). An optional solution to that is configuring git to automatically update submodules whenever you change branches. You can run this command once to change git behavior
`git config --global submodule.recurse true`

*This is a very quick overview of git submodule usage and by no means proper training. I suggest playing with it a bit to familiarize and reading more thoroughly about the usage before fully adopting it.

## Bottom Line

...so when to use each approach? It's mainly a question of **how frequently you plan to change it**. Let's sum it up:

-   If your use case enables you to **avoid shared code** without adopting any bad practices, this will probably be the way to go as it can **reduce complexity and cost**.
-   If you do have to share some code, but it's small and static enough so you'd never have to change it  - **consider duplicating it**. As bad practice as it is I've seen some cases that were simple enough that going into the bullets below was not a valuable choice.
-   If you still wish to share code - ask yourself **how frequently do you plan to change it**. If you're not going to change it regularly - using the **package manager** sounds like a good option.
-   If you wish to share code and you plan to make frequent changes to it  - **consider git submodules**.

**At [HUMAN](https://www.humansecurity.com/), we use both**. Shared libraries tend to have maturity factors. New libraries usually require somewhat frequent additions and bug fixes and we find it most effective to work with git submodules. Older, more stable libraries usually require much less frequent changes and these changes usually go through a more structured review/release processes. This fits well within the approach of the **package manager**.
Additionally, we share models and entities. As part of the business logic, these entities tend to change alongside the business logic of the application. With that, the development cycle of **going through the package manager was devastating**. We do, however, identify an ***owner*** repo to each entity, the owner is likely to perform changes to the entity itself and so it makes sense for it to consume the entity repo as a **git submodule**. Other applications might rely on this entity as well, but since they're not likely to perform changes to it, it **makes sense to consume it via the package manager**.

Next, we'll be discussing testing strategies in the [last part of this series]({% post_url 2022-05-08-testing-strategies-in-microservices-architecture %}).

## What Have I Missed?

Have I missed anything? **Please share your thoughts** and your approach with us! What do you think of ours? ❤️

Special thanks to my superstar colleague [Vlad Mayakov](https://www.linkedin.com/in/mayakov-vlad/) for helping in making sense and putting it all together.