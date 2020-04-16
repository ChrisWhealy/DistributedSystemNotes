# Distributed Systems Lecture 3

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 3rd, 2020 via [YouTube](https://www.youtube.com/watch?v=83Ha1rX2LSw)

## Lamport Diagrams, Causality and "happens before"

Recap : the notation `A -> B` means *"A happens before B"*

This means that `A` could have been the cause of `B`, but `B` ***cannot*** be the cause of `A`

However, when representing this information, we need a notation that makes time explicit

## Lamport Diagrams (a.k.a. Spacetime Diagrams)

Draw a vertical line to represent the events that happen within a process.  (Time moves in the downwards direction).

Events are then represented as dots on that line.

![Process events](./img/L3%20Process%20events.png)

This diagram tells us that three events have taken place within this process and that they happend in the order `X` followed by `Y` followed by `Z`.

From this we can infer that event `X` ***might*** be the cause of events `Y` or `Z`, but we can't say that for certain.  All we know is that event `X` happened before either of the events `Y` or `Z`

In the case of multiple machiens, we would represent these as a set of vertical lines, each with their own timeline of events

![Multiple processes](./img/L3%20Multiple%20Processes.png)

### Communication Between Machines

The only way these machines can communicate with each other is by sending messages.  The send and receive events then show up as dots on each machine's time line.

![Message passing between processes](./img/L3%20Message%20Passing.png)

Generally speaking, given two events, `A` and `B`, we can say that `A` happens before `B` (`A -> B`) if any of the following are true:

- Events `A` and `B` are events in the same process and `B` happens after `A`
- If `A` is a message send event and `B` is the coresponding receive event, than `B` must have happend after `A` because it makes no sense to talk of a message being received ***before*** it was sent
- If `A -> C` and `C -> B`, the we can be certain that `A -> B` (This is know as transitive closure)

This is the definition of the ***"happens before"*** relation.

## Causal Anomalies

Here's an example that highlights the importance of needing to know the exact order in which events occurred.  Without this knowledge, you might well be left wondering why you've received a particular message.

![A causal anomaly](./img/L3%20Causal%20Anomaly.png)

1. Alice sends a message to both Bob and Carol saying *"Bob Smells"*
1. Bob takes offence and immediately responds to both Alice and Carol by saying *"Up yours!"*
1. However, Alice's original message to Carol is delayed for some reason, and results in Bob's rude response arriving ***before*** Alice's original message

Without a clear understanding of concept of the "happens before", Carol will not be able to understand why Bob has sent her such a rude message.

## Network Models

### Synchronous or Asynchronous?

If we could say with certainty that sending a network message required no more than `N` units of time, then we would have a much better idea of how communication should be managed.  Under these circumstances, using timeouts to reason about communication failure would be a reasonble approach.  Networks that make such guarantees are known as "synchronous networks" (E.G a network in which we know that message transmission will take no more than `N` units of time).  In general, synchronous networks require the existence of a stable circuit between sender and receiver.

However, this is not how the internet works.  The internet is an asynchronous network (I.E. an network having no central point of control and in which there is no upper bound on message transmission time)

As an aside, there is also a type of network known as *"partially synchronous network"* in which the upper bound on transmission time is large, but finite. (For details, see the book [Distributed Algorithms](https://www.amazon.co.uk/Distributed-Algorithms-Kaufmann-Management-Systems-ebook/dp/B006QUTUR2/ref=sr_1_1) by Nancy Lynch)

### But Can We Reason About Transmission Times?

Well, it depends on the type of network you're using...

The least forgiving network model is the asynchronous one.  By "unforgiving", we mean a network that makes the least assumptions about message transmission and allows us the least scope for reasoning about its behaviour.

In spite of it being unforgiving, the most robust designs are built on asynchronous networks.  The flip side of this however is that these are also the hardest networks to reason about.  One consequence being that you will be unable to prove that certain types of situation or event are impossible.

If on the other hand, you want to prove than a certain type of event is impossible, then you should choose the most forgiving network model (the synchronous one), for if you can prove that a certain event is impossible in the most forgiving network (for instance, where message delivery is known never to exceed `N` units of time), then you can also be certain that the same event will be impossible in the least forgiving network where `N` is unbounded.

Here, a "forgiving network protocol" is one that makes the most assumptions and allow us the greatest scope for reasoning about its behaviour


## State

So far, our Lamport diagrams only show events and the transmission of messages between processes.  How then should state be described?

### What is the 'State' of a Computer?

The term "state" is typically understood to mean the contents of a computer's memory and registers at some particular point in time.

So if at some particular point in time, `X=5`, then it follows that there must have been some previous point in time when `X` did not equal `5`.  The transistion from `X` not equalling `5` to `X` equalling `5` is an event that can be represented on a Lamport diagram.  So events also represent state.

So some would propose that we should be able to reconstruct the machine's state at any point in time if we know two things:

1. The *full* history of all events that led up to `X` becoming equal to `5`
1. The precise order in which those events occurred

In reality however, even if we have this knowledge, it might still not be possible to reconstruct the state.

### Reasoning About State

Recap: There are three different ways in which events can be in the `->` "happens before" relation

1. Events `A` and `B` occur in the same process, with `B` happeining after `A`
1. If `A` is a message send event and `B` is the corresponding receive event, then `B` ***must*** happen after `A` because a receive event ***cannot*** happen before its corresponding send event
1. If `A -> C` and `C -> B`, then we can be certain that `A -> B` (Transitive closure)


So if `A -> B`, then events `A` and `B` form a pair of events ordered by the **"happens before"** relation.

![Reasoning about State](./img/L3%20Reasoning%20About%20State.png)

What can we say about the ordering of these events?

* From rule 1: `B -> C`
* From rule 2: `A -> B` and `D -> C`
* From rule 3: `A -> C`

***Q:***&nbsp;&nbsp; What can we say about the order of events `A` and `B` in relation to event `D`?  
***A:***&nbsp;&nbsp; Nothing!


So the state of the machine is represented by the smallest set of ordered pairs, that is:

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
                      
Does `P` happen before `S`?  
Yes, `P` is a send event and `S` is the corresponding receive event, therefore `P -> S`

What about `X` and `Z`?  
Yes, events `X` and `Z` occur within the same process and `Z` happens after `X`, therefore `X -> Z`

What about `P` and `Z`?  
Yes, `P -> S`, and `S` and `Z` are in the same process with `Z` happening after `S`, therefore `P -> Z`

What about `Q` and `R`?  
We do know that `Q` was caused by `T` and that `T -> Z` and `Z -> U` and `U -> R`, but since we are not allowed to determine causality by travelling backwards in time, we must conclude that `Q` and `R` are ***not*** related by the "happens before" relation.  In spite of the visual position of the dots in the diagram, we are unable to say which event happened first.

Another way of saying this is that the ***"happens before"*** relation cannot be used to form an ordered pair from `Q` and `R`.  All we can say about `Q` and `R` is that these events are **"concurrent"** or **"independent"**.  This is written as `Q || R`.

In the above diagram, events `X` and `P`, and `Z`, `U`, `V` and `Q` are also concurrent.


## Partial Orders

A partial order is where, for a given set `S`, a binary relation holds that exhibits the following properties.  This binary relation is usually, but not always written as `≤` (less than or equals) and allows you to compare the members of `S` using the following properties:

| Property | English Description | Mathematical Description |
|---|---|---|
| Reflexivity   | For all `a` in `S`<br>`a` is always `≤` to itself | `∀ a ∈ S: a ≤ a`
| Anti-symmetry | For all `a` and `b` in `S`<br>if `a ≤ b` and `b ≤ a`, then `a = b` | `∀ a, b ∈ S: a ≤ b, b ≤ a => a = b`
| Transitivity  | For all `a`, `b` and `c` in `S`<br>if `a ≤ b` and `b ≤ c`, then `a ≤ c` | `∀ a, b, c ∈ S: a ≤ b, b ≤ c => a ≤ c`

However, it makes no sense to say that event `A` ***"happens before"*** itself, so when speaking of a set of events, the reflexivity property is nonsensical and therefore not applicable.

Also, we will never encounter the situation in a distributed system where event `A` happens before event `B` ***and*** event `B` happens before event `A` (making `A` and `B` the same event).

The anti-symmetry property is never adhered to in real life, but strictly speaking, it is still never violated; therefore, this rule is said to be ***"vacuously true"***.  That is, when dealing with a set of real-life events, we will never find an example that exhibits this property; however, we will also never find an example that violates this property either...

So the "happens before" relation is a weird kind of partial order because only two of the three rules of a partial order apply, and even then, one of the two is only vacuously true.

Therefore, the ***happens before*** relation is said to be *"an irreflexive partial order"*


In distributed systems, we will be dealing with many different kinds of partial order.  It is strange however that the first partial order we encounter is "happens before", but this is because it is both fundamental to distributed systems, but also a weird kind of partial order.


Whenever we talk about a relation being a partial order, we must first look at the set of things we're dealing with.  If we're dealing with the "happens before" relation, then we're dealing with a set of events












