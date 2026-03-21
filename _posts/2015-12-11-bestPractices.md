---
layout: post
title: "Coding Best Practices"
---

<img src="{{ '/images/CodingBestPractices.jpg' | relative_url }}" alt="Coding best practices cover image">

This is an older set of notes I wrote down for myself on coding habits that still matter.

None of this is especially groundbreaking, but these are the kinds of things that usually make a bigger difference than whatever framework or language happens to be popular at the moment.

## What I Still Believe

* Build the simplest version first. An MVP is usually the fastest way to learn whether your idea actually works.
* Before writing code, spend a little time thinking through the shape of the solution. Even rough pseudocode saves a lot of wasted effort.
* When a problem feels too big, break it into smaller pieces and solve one part at a time.
* Error messages are useful. If the output still does not make sense, debugging tools are usually more helpful than random code changes.
* When debugging, try to form a theory and test it. Moving code around without a reason usually just creates more confusion.
* Develop a workflow that helps you make steady progress. For me, that usually means committing changes in reasonable chunks.
* Learn the shortcuts and tools in your editor well enough that they stop getting in your way.

## Design Habits

* Modularity matters. Splitting code into smaller pieces usually makes it easier to reason about and easier to change later.
* Loose coupling matters. The less one part of the codebase depends on another, the easier it is to evolve both.
* Thin interfaces are underrated. Functions that take a small number of inputs are usually easier to understand and test.
* Intuitive naming is worth the effort. Good names reduce the amount of mental overhead every time you come back to the code.
* Passing dependencies explicitly is usually cleaner than quietly depending on globals or hidden state.
* Avoid unnecessary side effects when you can. Predictable code is easier to debug and easier to trust.

## Cleanup Matters

* Once the basic logic is working, clean it up. Shorter, clearer code is usually easier to maintain.
* DRY still matters, but only after the pattern is real. I would rather duplicate something twice than abstract it badly too early.
* Stay humble and keep learning. This field changes too fast for ego to be useful for very long.
