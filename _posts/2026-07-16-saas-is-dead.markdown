---
date: 2026-07-16 00:00:00
title: 'SaaS *is* dead. But not because AI "killed" it, because AI killed it.'
description: SaaS is dead, and vibe coding killed it from within. Why today's coding agents are built for vibe coders, not the software engineers who actually pay for them.
image: images/posts/saas-is-dead/saas-is-dead.jpg
categories: [coding-agents, llm]
tags: [coding-agents, llm]
series: agentic-control-for-software-engineers
---

<div class="series-nav">
  <div class="series-nav__title">Agentic Control for Software Engineers</div>
  <div class="series-nav__subtitle">Article 1 of 2</div>
  <div class="series-nav__list">
    <span class="series-nav__item--current">1. SaaS Is Dead ← You are here</span>
    <a href="/agentic-control-for-software-engineers" class="series-nav__item">2. Agentic Control for Software Engineers</a>
  </div>
</div>

Normally, having to explain a joke is a bad sign, but since you couldn't see my facial expressions while typing this title, I might have to explain what I meant. And why this is hilariously ironic.

Thing is, all those Anthropic news reporters (roughly every other person who posted on LinkedIn in '25-'26) claimed "SaaS is dead" so many times, each time with a different "this is the end of the world as we know it" explosive title, where the underlying statement is always the same: vibe coding has made SaaS irrelevant. Yet again. You never need to pay for online services again. Need something? A 10-minute vibe coding session will solve it for you and save you a lot of money. "SaaS is dead", they confidently declared.

6-18 months into this now. SaaS is dead. Vibe coding did kill it. Not by replacing every online service with a secure, robust, custom solution tailor-made to the specific needs of each person on earth (boy would that be wonderful. Also, boy how naive was I to give them the benefit of the doubt). Vibe coding simply killed it from within. That's the first irony. Have you seen the GitHub status page lately?

![Shoutout to @anshuc on Twitter for this punchline I literally LOLed at: "how did the attackers find a large enough uptime window to get in?"](/images/posts/saas-is-dead/github-uptime-roast.png)

We've gotten used to SaaS solutions taking pride in their five-nines stats back in '22, being down once a week, at best, in '26. What the heck happened?

Well, the answer is kind of obvious. But to go a bit deeper beyond the obvious surface, the interesting thing is the radical shift in expectations of markets of the software industry. As time goes by, it's becoming clearer to us that AGI is not around the corner, that vibe coding is not going to replace software engineers, and that it's not going to accelerate software development 10x. However, the markets don't really respond to that part. They've already invested in seeing this happen. People have already been laid off, financial forecasting has already been presented to the boards, and now we already *have* to deliver 10x. Somehow. I'll leave the rest of the math to you.

Don't get me wrong, I'm very much in favor of assisted coding. As someone who considers himself a person who extremely enjoys the practice of programming itself and had initial fears, I quickly discovered how many different layers of programming give me joy and how much of it I can automate and still love every second. I'm probably at the higher range of AI adoption, and relative to someone who used to code many hours of every single day, I now rarely ever write code directly.

However, these tools are not made for us, software engineers. We need more control. We need better tools around the LLM. Why am I claiming it with such confidence? Let me give you some examples.

## Human _Not_ in the Loop
Have you ever given a coding task to an agent, come back after a few minutes and found a clean branch with committed code? I know, this is not the end of the world. You can fix the code, you can amend the commit. But the itch to get mad at the agent is there. Why? Because you're the one who has to answer for issues in the shipped code. Yet, nobody thought about looping you in. That's annoying, and let's admit it: 9 times out of 10, the code is not going to fully match your intent on that first iteration. That's not _just_ annoying. It's also counterproductive.

## Rules Are Meant to Be Broken
Have you found yourself begging the agent to follow your coding standards? You find yourself inflating your AGENTS.md file, repeating rules like: "you must always handle errors properly", "never place tests in that directory", "never attempt to use AWS CLI directly, we work with IaC!!!". Did it work? It might have helped a bit, and probably made other things worse. Have you ever tried adding rules instructing the agent to write fewer comments? Try it out. See how it goes 🤦‍♀️. Nothing is consistent, nothing is deterministic. What else do we have left but yelling at it "BUT YOU'RE NOT SUPPOSED TO DO THAT!".

## You're a Senior Software Engineer
Do you work with Claude Code? Have you ever found yourself asking it to investigate something and provide an analysis? Have you then gotten tired of hitting Enter aimlessly and switched to auto mode? Did you come back 5 minutes later to discover it committed "the fix" although you clearly asked for a root cause analysis!!! What fix? What was the issue, for crying out loud?!?! You know why that is? Claude Code conflates the agentic control (the harness mode, the thing that automatically approves tool calls) with the system prompt. For some reason somebody thought something like this: "this person doesn't want to keep hitting Enter. It probably means we should tell the LLM it is a _senior_ software engineer (🤣) and please strive to achieve a solution". So it skips the root cause analysis you requested. It goes directly to implementing things you *did not* request. There it is again: your intent drifting away.

## The Takeaway
What do all of these stories hold in common? Lack of control. Missing tools to better convey your intentions and make sure they're kept. Inability to enforce rules and standards. I can go on and on. The point is _why_ it happens.

It happens simply because these tools are meant for vibe coders. They're clearly built for someone who has no intention of looking at the code, enforcing standards, or being kept in the loop. These tools are built on the premise of AGI, where you don't really need software engineers. Everyone can code while vibing and enjoying a drink on a beach with their laptops. Stop and enjoy the moment, dudddde. 🌸🍹🏖️💻 Software is famously solved. 👍

And here comes the second irony: Anthropic is one of the fastest-growing companies in history. Where is all that money coming from? Vibe coders? Single developers with no funds who want to build an alternative dating app? The money is clearly coming from enterprise companies where the vast majority of users are professional software engineers who suffer from lack of capabilities and control. Yet, the focus is still vibe coding.

So long as responsibility for what we ship is on us, we need to be in control.

Get in touch with me ([LinkedIn](https://www.linkedin.com/in/avivcarmis), [email](mailto:avivcarmis@gmail.com)) if you want to get some details about a product we built aiming at giving control back to the engineers. Follow up to [the next post](/agentic-control-for-software-engineers) to hear more about what it does.

Regardless of any specific solution, the movement should be clear. We need to gain control to be able to ship fast without breaking things. The future is not in vibe coding. I mean...it's kind of like...I'm sorry I can't help myself...vibe coding is dead?
