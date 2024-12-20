---
title: What the F*#% is This Code Doing?
---

I have read far too much code that has made me stop and think, in full rational thought: "what the actual F is this code doing!?" In all sense of the phrase. Why does this code exist? What does it achieve? What are the characters on the screen doing? What *is* this code? Code that is catastrophically underdocumented. It's a disease, and it's everywhere.

The company I work for is particularly bad in the documentation regard, and is sourcing my annoyance to write this article today. I spend far too much time (and my colleagues' time) trying to piece together meaning in codebases with near-zero context provided. I mean it's really egregious. Repos with hundreds of files, tens of thousands of commits, dozens of contributors, that have: meaningless names, no readme, no design documentation, no inline comments, and useless commit messages. These repos pull you so far down into the abyss that it will make your day to find a single comment or person relevant to the feature you're interested in. You could spend hours reading the code and gather little-to-no intuition about what the code is doing. No joke, the most difficult part of making changes to these systems is simply knowing what the heck you're looking at and where to slot your code in. Of course, the documentation problem is not limited to my workplace. Open source can be just as bad and I'm sure other companies are too. It astounds me how large projects can grow, yet simultaneously be so damn inaccessible. The whole thing is plain stinky. Lousy project ownership and shoddy engineering. Take a minute to say a prayer for all the new starters out there who are pushed through this dumpster fire and expected to deliver excellence.

My message today is: please, for the love of everyone on this planet, document your code. No, it's not that hard. Here's how to do it.

## What is This Software?

The first thing any codebase needs is something outlining what it is and why it exists, at a high level. If I happen across your repo, please tell me what I'm looking at and how it's used. How am I expected to interact with a codebase if I don't know anything about its purpose?

Some questions to answer in this high-level documentation:

- Why does the software exist?
- What problem does the software solve?
- What are the goals/priorities of the software?
- Where/how is the software deployed and used?
- How do I build the software?
- How do I run the software?

These should be answered from a reasonably general perspective if possible, not 25 layers down the rabbithole in a domain so specific only one person understands it. If useful, link to other projects, code, or documentation to help the reader gather context.

If a new developer comes to the codebase, reads this high-level documentation, and still has no clue what is going on, the documentation has **failed**.

## Design Documentation

An important piece of context often missing is explanation of how the code is structured to achieve the software's goals - i.e. design. There are a million and one ways to churn out code that creates some software. Why did you pick this specific way? Answering this dilemma is one of the core, never-ending challenges of software development, so write it down, damn it! It's the critical bridge from software, the concept, to software, the source code.

Some questions to answer in design documentation:

- How is the domain problem broken down into modules/services?
- How do modules/services interact?
- What commonplace patterns does the design follow?
- What oddities are there in the design?
- What technologies support the design?
- How are security, reliability, and performance approached?

And, for each, explain *why*! What influenced the design decisions? Why is the design not something else? I can probably think of five "better" ways to design your software, so convince me why I'm wrong. Make me confident to modify the codebase, knowing a solid design is backing us up.

## Implementation Documentation

Some people claim source code can be self-documenting. I strongly disagree. Developers work at the level of source code, thus we need documentation there. Inline code comments are like little hits of sanity to keep us on track, nudging us towards higher reasoning, preventing the limitless complexity of modern languages from overwhelming our mental working space. I don't want to have to sit there and reverse-engineer every line of your code, thinking through every "what if?", to link to higher meaning. Code itself should be clean, but if that fails, inline comments are the next line of defense. *Good* comments, that is.

Good inline comments answer questions like these:

- What does the file/class/function/constant/object represent, conceptually?
- What does the function input and output, conceptually?
- What conditions does the code assume?
- What are the edge cases and gotchas in the code's behaviour?
- When can the code fail?
- What happens if the code fails?
- Why does the code go against common practice?
- How does the code relate to other code elsewhere?

A natural language description of what the code directly implements is not helpful, unless the code is exceptionally technical within the reasonable context of the reader. More important is linking to higher understanding of where the code fits in the purpose of the application. Particularly high-value comments explain why code exists, where it is not immediately obvious from the surrounding code.

## Change Documentation

Finally, put some effort into your commit messages and pull request descriptions. The kicker here is, again, say *why* the change is happening. Why, why, why. Worse than not knowing how code works is also not knowing why it was put there in the first place, and the author left the company seven years ago, and all you have to go on is a one-liner wimp of a commit message like "improve algorithm". For bonus points, link to other code, changes, and documentation that are necessary to comprehensively understand the change we are looking at. Not only will it help someone down the line, but your colleagues will love you for presenting them with nice pull requests to review.

## Do It

That's all. Document your damn code. Self-documenting software doesn't exist. Tell me what the f*#% I'm looking at.
