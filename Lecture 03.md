# Distributed Systems Lecture 3

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 3<sup>rd</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=83Ha1rX2LSw)

| Previous | Next
|---|---
| [Lecture 2](./Lecture%2002.md) | [Lecture 4](./Lecture%2004.md)

## Causality and the "Happens Before" Relation

The notation `A -> B` means *"A happens before B"* and helps make the ordering of events in time explicit.

The "happens before" relation allows us to conclude two things:

1. It is possible that event `A` ***might*** have been the cause of event `B`
1. It is ***completely impossible*** for event `B` to have been the cause of event `A`

In other words, the arrow in the happens before relation indicates the direction of possible causality.

## Lamport Diagrams (a.k.a. Spacetime Diagrams)

With time moving downwards, draw a vertical line to represent the events that happen within a process.<sup id="a1">[1](#f1)</sup>
Events are then represented as dots on that line.

![Process events](./img/L3%20Process%20events.png)

This diagram tells us that three events have taken place within this process and that they happened in the order `X` followed by `Y` followed by `Z`.
From this, we can then infer the following:

* Event `X` happened before events `Y` and `Z`, and therefore ***might*** be their cause, but we cannot be certain about this
* Event `Y` happened before event `Z` and therefore ***might*** be its cause, but again, we cannot be certain about this
* Events `Y` and `Z` happened after event `X`. We can therefore say with 100% certainty that `X` was not caused by either `Y` or `Z`
* Event `Z` happened after event `Y`.  We can therefore say with 100% certainty that `Y` was not caused by `Z`

In the case of multiple machines, we would represent these as a set of adjacent vertical lines, each with their own timeline of events.

![Multiple processes](./img/L3%20Multiple%20Processes.png)

### Communication Between Machines

The only way these machines can communicate with each other is by sending messages.
The send and receive events are represented as dots on each machine's timeline.

![Message passing between processes](./img/L3%20Message%20Passing.png)

Generally speaking, given two events, `A` and `B`, we can say that `A` happens before `B` (`A -> B`) if any of the following are true:

- Events `A` and `B` are events in the same process and `B` happens after `A`
- If `A` is a message send event and `B` is the corresponding receive event, then `B` must have happened after `A`.  
    Sorry kids, time travel is not possible, so it makes no sense to talk of a message being received ***before*** it was sent
- If `A -> C` and `C -> B`, the we can be certain that `A -> B` (This is known as transitive closure)

This is the definition of the ***"happens before"*** relation.

## Causal Anomalies

Here's an example that highlights the importance of needing to know the exact order in which events occurred.
Without this knowledge, you might well be left wondering why you've received a particular message.

![A causal anomaly](./img/L3%20Causal%20Anomaly.png)

1. Alice thinks Bob has a problem with personal hygiene and sends a message to both Bob and Carol saying *"Bob Smells"*
1. Bob takes offence and immediately responds to both Alice and Carol by saying *"Up yours!"*
1. However, Alice's original message to Carol is delayed for some reason, and results in Bob's rude response arriving ***before*** Alice's original message

Without a clear understanding of concept of the "happens before", Carol will not understand why Bob has apparently started sending her insulting messages.

## Network Models

### How Long Does it Take to Send a Message?<br>(Would that be Synchronous or Asynchronous?)

If we could say with certainty that sending a network message required no more than `N` units of time, then we would have a much better idea of how communication should be managed.
If such an upper limit could be placed on communication performance, then using timeouts to reason about communication failure would be a reasonable approach.

Networks that make such timing guarantees are known as "synchronous networks" (E.G a network in which we know that message transmission will take no more than `N` units of time).
In general however, synchronous networks require the existence of a stable circuit between sender and receiver &mdash; which is exactly what does not exist in either public switched telephone neworks ([PSTNs](https://en.wikipedia.org/wiki/Public_switched_telephone_network)) or the internet.

The internet is an asynchronous network (I.E. a network having no central point of control and in which there is no upper bound on message transmission time)

> As an aside, there is also a type of network known as *"partially synchronous network"* in which the upper bound on transmission time is large, but finite.
> (For details, see the book [Distributed Algorithms](https://www.amazon.co.uk/Distributed-Algorithms-Kaufmann-Management-Systems-ebook/dp/B006QUTUR2/ref=sr_1_1) by Nancy Lynch)

### But Can We Reason About Transmission Times?

Well, it depends on the type of network you're using...

The least forgiving network model is the asynchronous one.  By *"least forgiving"*, we mean a network that:

*  Makes the least number of assumptions about message transmission, and
*  Allows us the least scope for reasoning about its behaviour

In spite of it being so unforgiving, the most robust designs are built on asynchronous networks.
The flip side of this however is that these are also the hardest networks to reason about.
One of the key consequences here is that we have no ability to describe all possible behaviours our system might exhibit, and without this ability, we can only protect our system against ***some*** of the possible error conditions, not all.

If, on the other hand, you want to prove than a certain type of event is impossible, then you should choose the most forgiving network model (the synchronous one); for if you can prove that a certain event is impossible in the most forgiving network (for instance, where message delivery is known never to exceed `N` units of time), then you can also be certain that the same event will be impossible in the least forgiving network where `N` is unbounded.

Here, a *"forgiving network protocol"* is one that makes the most assumptions and allow us the greatest scope for reasoning about its behaviour

## State

So far, our Lamport diagrams only show events and the transmission of messages between processes.
How then should state be described?

### What is the 'State' of a Computer?

The term "state" is typically understood to mean the contents of a computer's storage (registers, memory and permanent storage) at some particular point in time.

So, if at some particular point in time, `x=5`, then it follows that there must have been some previous point in time when `x` did not equal `5`.
The transition from `x` not equalling `5` to `x` equalling `5` is an event that can be represented on a Lamport diagram.
Therefore, events can also be thought of as representing changes of state.

So, it might seem reasonable to propose that we should be able to reconstruct a machine's state at any point in time if we know two things:

1. The *full* history of all events that led up to `x` becoming equal to `5`
1. The precise order in which those events occurred

In reality however, even if we have this knowledge, it might still not be possible to reconstruct the state.

### Reasoning About State

There are three different ways in which events can be ordered using the `->` *"happens before"* relation:

1. Events `A` and `B` occur in the same process, with `B` happening after `A`
1. If `A` is a message send event and `B` is the corresponding receive event, then `B` ***must*** happen after `A` because it makes no sense to talk of a receive event happening ***before*** its corresponding send event
1. If `A -> C` and `C -> B`, then we can be certain that `A -> B` (Transitive closure)

So, if `A -> B`, then events `A` and `B` form a pair of events ordered by the **"happens before"** relation.

![Reasoning about State](./img/L3%20Reasoning%20About%20State.png)

What can we say about the ordering of these events?

* From rule 1: `B -> C`
* From rule 2: `A -> B` and `D -> C`
* From rule 3: `A -> C`

***Q:***&nbsp;&nbsp; What can we say about the order of events `A` and `B` in relation to event `D`?  
***A:***&nbsp;&nbsp; Absolutely nothing!

So, the state of the machine is represented by the smallest set of ordered pairs, that is:

```
 { (A,B)
  ,(B,C)
  ,(A,C)
  ,(D,C)
  }
```

This list includes only those event pairs that obey the ***happens before*** relation.

What then can we say about the events shown in the following diagram?

![Message passing between processes](./img/L3%20Message%20Passing.png)

***Q:***&nbsp;&nbsp; Does `P` happen before `S`?  
***A:***&nbsp;&nbsp; Yes, `P` is a send event and `S` is the corresponding receive event, therefore `P -> S`

***Q:***&nbsp;&nbsp; What about `X` and `Z`?  
***A:***&nbsp;&nbsp; Yes, events `X` and `Z` occur within the same process and `Z` happens after `X`, therefore `X -> Z`

***Q:***&nbsp;&nbsp; What about `P` and `Z`?  
***A:***&nbsp;&nbsp; Yes, `P -> S`, and `S` and `Z` are in the same process with `Z` happening after `S`, therefore `P -> Z`

***Q:***&nbsp;&nbsp; What about `Q` and `R`?  
***A:***&nbsp;&nbsp; We do know that `Q` was caused by `T` and that `T -> Z` and `Z -> U` and `U -> R`; however, we are not allowed to determine causality by travelling backwards in time, so we must conclude that `Q` and `R` are ***not*** related by the "happens before" relation.
In spite of the visual position of the dots in the diagram, we are unable to say which event happened first.

Another way of saying this is that the ***"happens before"*** relation cannot be used to form an ordered pair from `Q` and `R`.
All we can say about `Q` and `R` is that these events are **"concurrent"** or **"independent"**.  This is written as `Q || R`.

In the above diagram, events `X` and `P`, and `Z`, `U`, `V` and `Q` are also concurrent.


## Partial Orders

A partial order is where, for a given set `S`, a binary relation holds that exhibits the following properties.
This binary relation is usually, but not always written as `≤` (less than or equals) and allows you to compare two members of `S` using the following properties:

| Property | English Description | Mathematical Description |
|---|---|---|
| Reflexivity   | For all `a` in `S`,<br>`a` is always `≤` to itself | `∀ a ∈ S: a ≤ a`
| Anti-symmetry | For all `a` and `b` in `S`,<br>if `a ≤ b` and `b ≤ a`, then `a = b` | `∀ a, b ∈ S: a ≤ b, b ≤ a => a = b`
| Transitivity  | For all `a`, `b` and `c` in `S`,<br>if `a ≤ b` and `b ≤ c`, then `a ≤ c` | `∀ a, b, c ∈ S: a ≤ b, b ≤ c => a ≤ c`

However, when the members of the set are events whose ordering is determined by some measure of time, it makes no sense to say that event `A` ***"happens before"*** itself; so when speaking of a set of events, the reflexivity property is nonsensical and therefore not applicable.

Also, we will never encounter the situation in a distributed system where event `A` happens before event `B` ***and*** event `B` happens before event `A` (making `A` and `B` the same event).
Thus, the anti-symmetry property can never be adhered to in real life.
Strictly speaking however, whilst this rule is never followed, it is also never violated; therefore, this rule is said to be ***"vacuously true"***.
That is, when dealing with a set of real-life events, we will never find an example that exhibits this property; however, we will also never find an example that violates this property...

So, the "happens before" relation is a weird kind of partial order because only two of the three rules governing partial orders apply, and even then, one of those two is true only in a vacuous sense.

Therefore, the ***happens before*** relation is said to be *"an irreflexive partial order"*.

In distributed systems, we will be dealing with many different kinds of partial order.
So the first fundamental principle we find governing the behaviour of distributed systems is this weird partial order called "happens before".

Whenever we talk about a relation being a partial order, we must first look at the set of things we're dealing with.  If we're dealing with the "happens before" relation, then we're dealing with a set of events.

---

| Previous | Next
|---|---
| [Lecture 2](./Lecture%2002.md) | [Lecture 4](./Lecture%2004.md)

---

***Endnotes***

<b id="f1">1</b>&nbsp;&nbsp; It is not a requirement for the direction of time to be downwards in a Lamport Diagram.
This is simply a stylistic choice; however, time is most often drawn moving either downwards, or from left to right.

[↩](#a1)

