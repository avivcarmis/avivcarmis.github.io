---
date: 2022-05-08 18:44:25
title: Testing Strategies in Microservices Architecture
description: Automated testing in microservices architecture
image: images/posts/testing-strategies-in-microservices-architecture/testing-strategies-in-microservices-architecture.webp
categories: [microservices, architecture, developer-experience, testing]
tags: [microservices, architecture, developer-experience, testing, rollout]
series: developer-experience-in-microservices
---
TL;DR - this article discusses automated testing in microservices architecture, how to keep it as simple as possible, and how to gain maximum reliability during the rollout of big changes. 👩‍🎓

\* This is part 3 of a 3 piece article series discussing key decisions in designing developer experience and development cycle for teams in a microservices architecture. 💡🚀
- [Designing Developer Experience in Microservices Architecture (intro)]({% post_url 2022-04-21-designing-developer-experience-in-microservices-architecture %})
- [Version Control Layout in Microservices Architecture (pt. I)]({% post_url 2022-04-29-version-control-layout-in-microservices-architecture %})
- [Code Sharing in Microservices Architecture (pt. II)]({% post_url 2022-05-03-code-sharing-in-microservices-architecture %})
- **Testing Strategies in Microservices Architecture (pt. III)**

## Testing Strategies

Integration testing or end-to-end testing in microservices architecture can get very complex very fast. I've seen countless cases of frustration and time-consuming efforts to pass such tests in automated environments, and I believe the first strategic decision is how much are we willing to sacrifice velocity in order to get stronger assurances and validation before going live. How much are we willing to increase the time and complexity of our build process, in order to increase reliability. 📈 📉

I'm not writing this piece to discuss the importance of testing, nor what areas of the code should be covered. If you've written integration or end-to-end tests you'd probably want to execute them to validate the whole system and catch degradations before going live. I want to argue that in distributed microservices architecture these kinds of tests offer yet another important value - rollout plan validations.

## Consistent Version Guarantee

In the series intro, we've defined [consistent version guarantee]({% post_url 2022-04-21-designing-developer-experience-in-microservices-architecture %}#consistent-version-guarantee) between components A and B as:

> Each version of component A is guaranteed to interact exclusively with a certain version of component B.

(If you haven't already, [quickly view the intro section]({% post_url 2022-04-21-designing-developer-experience-in-microservices-architecture %}#consistent-version-guarantee) to better understand guaranteed version interaction and to view some examples).

## Choosing a Testing Strategy

If you're using integration or end-to-end tests you'll probably gain the most value by matching your testing strategy to production architecture. For a consistent version environment, a [monorepo]({% post_url 2022-04-29-version-control-layout-in-microservices-architecture %}#monorepo) with such tests inside would provide strong enough validations while remaining simple. This can save hours of frustration and additional costs. 💪💰 Changes across several components can be done simultaneously, all components integration tested together at their latest version, exactly how they're guaranteed to interact in production. ([A hybrid repository layout]({% post_url 2022-04-29-version-control-layout-in-microservices-architecture %}#hybrid-solutions) for testing purposes can also be a good solution here).

For inconsistent version environments, things need to get a bit more complex. Let's discuss why. 🤔

## Breaking Changes

In inconsistent version environments, safely introducing breaking changes may get complex and has to be carefully planned. During the deployment of such changes, components will interact with multiple versions of the deployed component simultaneously. In order for it not to break, we must ensure the deployed component is backward compatible, and the other components are future compatible. When such deployment is completed we can go ahead and deploy the next component. Once all deployments are completed we can remove backward compatibility and the breaking change is fully deployed. 😵 😓

Rollout plans of breaking changes can be hard to design and harder to enforce. We could have made mistakes in the design, but even if the plan is impeccable, the execution of such a plan is highly prone to human error. 😮 😬

## Rollout Plan Validation

By using a [multirepo]({% post_url 2022-04-29-version-control-layout-in-microservices-architecture %}#multirepo) to capture inconsistent version production architecture, and using automated integration or end-to-end testing during our build process, we can safely validate each step of the plan before going live. 😌

So how do we do it? Actually, we already have! By capturing our production architecture using a multirepo layout and using a dedicated repository for our tests. Each time we merge changes in a component it's automatically being tested against the production version of all other components, if something breaks the tests, it will also break in production. Each time we must merge exactly one component, if it's not fully compatible with all other components it will break. When all tests have passed, we can go live, and then start working on deploying the next component. 👍

In a monorepo layout, we will test all components at their latest version. This will work well in test environments. However, since production environment is not version consistent to this exact combination - it might break during deployment and we're left without any automated process to validate it. 😱

I believe the type of assurance that multirepos provide is a strong and important feature to have in a distributed microservices architecture, especially during cross-component changes like these. I believe the monorepo approach, as tempting as it is, may reduce visibility and assurances over such migrations and rollout plans in these kinds of environments.

## Conclusion

For a consistent version environment, choosing a multirepo layout will overcomplicate the entire development cycle, leading to additional costs and training of engineers. A simple monorepo solution will work best. Regarding the other advantages of multirepos, they can be matched with [proper usage and tuning of various tools]({% post_url 2022-04-29-version-control-layout-in-microservices-architecture %}#other-considerations). 💪 😎

For an inconsistent version environment, choosing a monorepo layout will end up blinding us. It will reduce our ability to validate deployments in cross-component changes. Choosing a multirepo layout will complicate things, however, using the right tools for code sharing ([like discussed in the previous article]({% post_url 2022-05-03-code-sharing-in-microservices-architecture %})), searchability, and coding standards enforcement, in addition to proper training of the teams should help keep things smooth. 🤝 🤙

## What Have I Missed?

Have I missed anything? Please share your thoughts and let me know. ❤️

Special thanks to my superstar colleague [Vlad Mayakov](https://www.linkedin.com/in/mayakov-vlad/) for helping in making sense and putting it all together.