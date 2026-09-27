---
title: "TLA+"
date: "2023-03-10"
weight: 90
---

**TL;DR** TLA+ is a language used for **specifying** concurrent and distributed systems**.** It's useful for finding flaws in your high-level system design, before writing a single line of code. It's a tool of math or logic.

## How it works

In TLA+, the execution of the system is represented as a sequence of **state machine transitions**. You model your system with a number of variables and how the variables transition to their next state with the events.

Each event has an **enabling condition**. The enabling condition is like a triggering event for a state machine. Typically, a state machine has an external input that drives its transition. But here, the driving factors are the internal variables within the system. With these self-feeding triggering conditions, it becomes a global, self-running state machine. With logic, we can exhaust all possible states (when it's possible to do so) and examine the safety of the system. To limit the total number of states, we usually use a small model size (e.g. supposing that the system only has 3 nodes) but it is usually still helpful for identifying crucial problems.

Message sent within the system is represented as **a set of messages**. What's interesting about the set is that we only care about the total set of messages sent without worrying about their order. This is to model a realistic asynchronous message exchange system where messages drop and reordering can happen.

## Syntax

I'll need to dig into it more at some point. My impression so far

- In each of the state transitions, you must express how *every* variable in the system gets changed.
  - If you omit a variable, it means that the variable can be any value, which is not necessarily what you want. Usually, you want some variables to change while others remain unchanged.
- You can express logic like finding a max element in a set.

## More resource

Tutorials: <http://lamport.azurewebsites.net/video/videos.html>

[What Good Is Temporal Logic?](https://www.microsoft.com/en-us/research/uploads/prod/2016/12/What-Good-Is-Temporal-Logic.pdf)

[How Amazon Web Services Uses Formal Methods](https://cacm.acm.org/magazines/2015/4/184701-how-amazon-web-services-uses-formal-methods/abstract)
