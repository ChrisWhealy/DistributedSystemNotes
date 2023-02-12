# Distributed Systems Lecture 4

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 6<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=zQk7U6InXZs)

| Previous | Next
|---|---
| [Lecture 3](./Lecture%2003.md) | [Lecture 5](./Lecture%2005.md)

## Recap

### What is a Distributed System?

> ***[Martin Kleppman](https://martin.kleppmann.com/)'s definition***  
> A collection of computing nodes connected by a network and characterised by partial failure

> ***[Peter Alvaro](https://dl.acm.org/profile/81453654530)'s definition***  
>  A collection of computing nodes connected by a network and characterised by partial failure and unbounded latency


### What is the Definition of the *"Happens Before"* Relation

* Events `A` and `B` take place in the same process and `B` happens after `A`
* If `A` is a message send event and `B` is the corresponding receive event, then `B` ***cannot*** have happened before `A`.  
* If `A -> C` and `C -> B`, the we can be certain that `A -> B` (Transitive closure)


### The *"Happens Before"* Relation is an Irreflexive Partial Order

A partial order for a set `S` is where the members of `S` can be ordered using a binary relation such as *"less than or equal to"* `≤`.
The partial order then has the following properties:

| Property | English Description | Mathematical Description |
|---|---|---|
| Reflexivity   | For all `a` in `S`,<br>`a` is always `≤` to itself | `∀ a ∈ S: a ≤ a`
| Anti-symmetry | For all `a` and `b` in `S`,<br>if `a ≤ b` and `b ≤ a`, then `a = b` | `∀ a, b ∈ S: a ≤ b, b ≤ a => a = b`
| Transitivity  | For all `a`, `b` and `c` in `S`,<br>if `a ≤ b` and `b ≤ c`, then `a ≤ c` | `∀ a, b, c ∈ S: a ≤ b, b ≤ c => a ≤ c`

The set of all events in a system can be ordered by the *"happens before"* partial order; however, in the case of events, not all of the three properties described above are applicable.
It is impossible to apply the reflexivity property simply because it makes no sense to say that an event *happened before itself*.

### Accurate Terminology

Don't get hung up on thinking that you must always state that the "happens before" relation is an irreflexive partial order.
It’s fine just to call it a "partial order".

## What's an Example of a True Partial Order?

Set inclusion (or set containment).
This is defined as the set of all subsets of any given set.

So, if we have a set `{a, b, c}`, then the containment set `S` contains the following 8 elements; each of which is one of the possible subsets of `{a, b, c}` (Don't forget to include both the empty set and the entire set!):

```
{ {}
, {a}
, {b}
, {c}
, {a, b}
, {a, c}
, {b, c}
, {a, b, c}
}
```

All of these members are subsets of the original set and are related by the relation `⊆` (meaning "is a subset of") and can be represented as a lattice with the empty set at the bottom

![Partial order lattice](./img/L4%20Lattice.png)

The `⊆` relation ("is a subset of") is a true partial order because every element of the set adheres to the rules of:

* Reflexivity
* Antisymmetry
* Transitivity

## Total Orders

As an aside...

***Q:***&nbsp;&nbsp; In the above inclusion set `S`, how is the element `{a}` related to the element `{b,c}`?  
***A:***&nbsp;&nbsp; It's not!
In a set defined by a ***partial order***, it is not possible to relate every member of that set to every other member.
That's why its called a ***partial*** order!

Some orders however are able to relate every element in a set to every other element in that set.
These are known as ***total orders***.

A good example of a total order is the counting (or natural) numbers.
If our set is the natural numbers and the relation is `≤`, then our lattice diagram of the containment set is simply a straight line:

![Total order](./img/L4%20Natural%20Numbers.png)

This is because every natural number is comparable to every other natural number using the `≤` relation.

## How Can a Computer Determine the *"Happens Before"* Relation?

This question relates to clocks - specifically ***Logical Clocks***

> A logical clock is a very unusual type of clock because it can neither tell us the time of day, nor how large a time interval has elapsed between two events.  
> All a logical clock can tell us is the order in which events occurred.

### Lamport Clocks

The simplest type of logical clock is a ***Lamport Clock***.
In its most basic form, this is simply a counter assigned to an event.
The way we can reason about Lamport Clock values is by knowing that:

```
  if A -> B then LC(A) < LC(B)
```

In other words, if it is true that event `A` happened before event `B`, then we can be certain that the Lamport Clock value of event `A` will be smaller than the Lamport Clock value of event `B`.

It is not important to know the absolute Lamport Clock value of an event, because outside the context of the *"happens before"* relation, this value is meaningless.
And even in the context of the *"happens before"* relation, we must have at least two events in order to compare their Lamport Clock values.

In other words, Lamport Clocks mean nothing in themselves, but are consistent with causality.

### Assigning Lamport Clocks to Events

When assigning a value to a Lamport Clock, we need first to make a defining decision, and then follow a simple set of rules:

1. First, decide what constitutes *"an event"*.
The outcome of this decision then defines what our particular Lamport Clock will count.  
    In this particular case, we decide that both sending ***and*** receiving a message ***is*** counted as an event.
    (Some systems choose not to count *"message receives"* as events).
1. Every process has an integer counter, initially set to 0
1. On every event, the process increments its counter by 1
1. When a process sends a message, the current Lamport Clock value is included as metadata sent with the message payload
1. When receiving a message, the receiving process sets its Lamport Clock using the formula  
    `max(local counter, msg counter) + 1`

If we decide that a *"message receive"* is not counted as an event, then the above formula does not need to include the `+ 1`.

### Worked Example

We can see how this formula is applied in the following sequence of message send/receive events:

We have three processes `A`, `B` and `C` and the Lamport Clock values at the top of the process line indicate the state of the clock ***before*** the message send/receive event, and the Lamport Clock values at the bottom of the process line indicate the state of the clock ***after*** the send/receive event has been processed.

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%201.png)

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%202.png)

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%203.png)

As you can see, the value of each process' Lamport Clock increases monotonically (that is, the value only ever stays the same, or gets bigger &mdash; it can never get smaller!)

## Reasoning About Lamport Clock Values

We know that if `A` happens before `B` (`A -> B`) then we also know that `A`'s Lamport Clock value will be smaller than `B`'s Lamport Clock value (`LC(A) < LC(B)`).

But can we apply this logic the other way around?  If we know that `LC(A) < LC(B)` then does this prove that `A -> B`?

Actually, no it doesn't.  Consider this situation:

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%204.png)

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%205.png)

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%206.png)

Let's compare the Lamport Clock values of processes `A` and `B`

```
  LC(A) = 1
  LC(B) = 4
```

Since `LC(A) < LC(B)` does this prove that `E1 -> E2`?

Absolutely not!

Why?
Because we have no way to connect event `E1` with event `E2`.
These two events do not sit in the same process, neither is there a sequence of message send/receive events that allows us to draw a continuous line between them.
So, in this case, we are unable to use the ***happens before*** relation to define any causal relation between events `E1` and `E2`.
All we can say is that `E1` is independent of `E2`, or `E1 || E2`.

So, in plain language:

> If we are unable to trace an unbroken connection between two events, then we are unable to establish a causal relation between those events

Or to use fancier, academic language:

> Causality requires graph reachability moving forwards through spacetime

So, in summary, when we reason about Lamport Clock values, if we know that `A -> B`, then we can be sure that `LC(A) < LC(B)`.  In other words:

> Lamport Clocks are consistent with causality

However, simply knowing that `LC(A) < LC(B)` does not prove `A -> B`; this is because:

> Lamport Clocks do not characterise (or establish) causality

The above example is derived from the ["Holy Grail"](./papers/holygrail.pdf) paper by Reinhard Schwarz and Friedemann Mattern.

### So, What Are Lamport Clocks Good For?

***Q:***&nbsp;&nbsp; Can we use a Lamport Clock to tell us the time of day?  
***A:***&nbsp;&nbsp; Nope

***Q:***&nbsp;&nbsp; Can we use a Lamport Clock to measure the time interval between two events?  
***A:***&nbsp;&nbsp; Nope

***Q:***&nbsp;&nbsp; So what are they good for?  
***A:***&nbsp;&nbsp; A Lamport Clock is designed to help establish event ordering in terms of the *happens before* relation

Even though Lamport Clocks cannot characterise causality, they are still very useful because they define the logical implication: `if A -> B then LC(A) < LC(B)`

All logical implications are constructed from a premise (`A -> B`) stated in the form of a question, and a conclusion (`LC(A) < LC(B)`) that can be reached if the answer to the question is true.

For any logical implication, we can also take its contra-positive.
Thus, if `P => Q` then the contra-positive states that `¬Q => ¬P`

So, in the case of the "happens before" relation

`if A -> B then LC(A) < LC(B)`

The contra-positive states

`if ¬(LC(A) < LC(B)) then ¬(A -> B)`

The contra-positive states that if the Lamport Clock of `A` is not less than the Lamport Clock of `B`, then `A` cannot have happened before `B`.
This turns out to be really valuable in debugging because if we know that event `A` ***did not*** happen before event `B`, then we can say with complete certainty that whatever consequences were created by event `A`, they could never have contributed to the earlier problem experienced during event `B`.


## Summary

Lamport Clocks are good for certain aspects of determining causality, but they do leave us with indeterminate situations.
This is because a Lamport Clock is consistent with causality but does not characterise it.

In order to remove this indeterminacy, we need a different type of clock, called a Vector Clock.

---

| Previous | Next
|---|---
| [Lecture 3](./Lecture%2003.md) | [Lecture 5](./Lecture%2005.md)
