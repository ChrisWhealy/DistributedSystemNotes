# Distributed Systems Lecture 4

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 6th, 2020 via [YouTube](https://www.youtube.com/watch?v=zQk7U6InXZs)

## Recap

### What is a distributed systems?

> ***Martin Kleppman's definition***  
> A collection of computing nodes connected by a network and characterised by partial failure

> ***Peter Alvaro's definition***  
>  A collection of computing nodes connected by a network and characterised by partial failure and unbounded latency


### What is the Definition of the *"happens before"* Relation

* If `A` and `B` are events in a single process and `B` happens after `A`, then `A -> B`
* If `A` is a message send event and `B` is the corresponding message receive event, then `B` *cannot* happen before `A`
* If `A` happens before `C` and `C` happens before `B`, then we know that `A -> B` (Transitive closure)


### The "happens before" Relation is an Irreflexive Partial Order

The set of all events in a system can be ordered by the "happens before" partial order, but this relation does not exhibit the reflexivity property, thus making it an irreflexive partial order.

A partial order for a set `S` is where the members of `S` can be ordered using a binary relation such as `≤`.  The partial order then has the following properties:

| Property | Description |
|---|---|
| Reflexivity   | `∀ a ∈ S: a ≤ a`
| Anti-symmetry | `∀ a, b ∈ S: if a ≤ b, b ≤ a then a = b`
| Transitivity  | `∀ a, b, c ∈ S: if a ≤ b, b ≤ c then a ≤ c`

But for the "happens before" relation, the property of reflexivity makes no sense, for an event cannot happen before itself

Don't get hung up on thinking that you must always state that the "happens before" relation is an irreflexive partial order.  Its fine just to call it a "partial order".



## What's an Example of a True Partial Order?

Set inclusion (or set containment).  This is defined as the set of all subsets of any given set

So if we have a set `{a, b, c}`, then the containment set `S` contains the following 8 elements; each of which is one of the possible subsets of `{a, b, c}` (Don't forget to include the empty set!):

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

The `⊆` relation ("is a subset of") is a true partial order because every element is the set adheres to the rules of:

* Reflexivity
* Antisymmetry
* Transitivity

## Total Orders

As an aisde...

- ***Question:***  
    In the above inclusion set `S`, how is the element `{a}` related to the element `{b,c}`?  
- ***Answer:***  
    It's not!   
    A set defined by a ***partial order*** cannot relate every member of that set to every other member. In fact, that's the point of it being called a ***partial*** order!

Some orders however are able to relate every element in a set to every other element in that set.  These are known as ***total orders***.

A good example of this is the natural numbers.  If our set is the natural numbers and the relation is `≤`, then our diagram of this set is simply a straight line:

![Total order](./img/L4%20Natural%20Numbers.png)

This is because every natural number is comprable to every other natural number using the `≤` relation.


## How Can A Computer Determine What's in the *"Happens Before"* Relation?

This question relates to clocks - specifically ***Logical Clocks***

A logical clock is a very unusual type of clock because it can neither tell us the time of day, nor how large the interval is between two events.  All a logical clock can do is tell us the order in which events occurred.

The simplest type of logical clock is a ***Lamport Clock***.  This is nothing more than a numerical value assigned to an event.  The way we can reason about Lamport clock values is by knowing:

```
  if A -> B then LC(A) < LC(B)
```

It is not important to know an event's Lamport Clock value because outside the context of the "happens before" relation this value is meaningless.  And even in the context of the "happens before" relation, we must have at least two events that are (potentially) causally related in order to compare their Lamport Clock values.

In other words, Lamport Clocks are consistent with causality.

### Assigning Lamport Clocks to Events

1. First decide what constitutes *"an event"*.  
    This decision then defines what our particular Lamport Clock will count.  In this particular case, we decide that receiving a message **is** counted as an event.  (Some systems choose not to count message receives as events).
1. Every process has an integer counter, initially set to 0
1. On every event, the process increments its counter by 1
1. When sending a message, a process includes the Lamport Clock value with the message
1. When receiving a message, set your Lamport Clock to `max(local counter, msg counter) + 1`.

If message receives are not counted as events, then the above algorithm does not need to include the `+ 1`

We can see how this algorithm is applied in the following sequences of message send/receive events:

We have three processes `A`, `B` and `C` and the Lamport Clock values at the top of the process line indicate the state of the clock before the message send/recdeive event, and the Lamport Clock values at the bottom of the process line indicate the state of the clock after the send/receive event has been processed.

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%201.png)

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%202.png)

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%203.png)


## Reasoning About Lamport Clock Values

We know that if `A -> B` then `LC(A) < LC(B)`

But what about the other way around?

If `LC(A) < LC(B)` then does this prove that `A -> B`?

Actually, no it doesn't.  Consider this situation:

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%204.png)

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%205.png)

![Lamport Clock Message Send 1](./img/L4%20LC%20Msg%20Send%206.png)


Let's compare the Lamport Clock values of processes `A` and `B`

```
  LC(A) = 1
  LB(B) = 4
```

Since `LC(A) < LC(B)` does this prove that `E1 -> E2`?

Absolutely not!

Why?  Because we cannot form a chain of connections between `E1` and `E2`.  These two events do not sit in the same process, neither is there a sequence of message send/receive events that allows us to draw a line between them

So in this case, the ***happens before*** relation is unable to define any causal relation between events `E1` and `E1`.  All we can say is that `E1` is independent of `E2`, or `E1 || E2`

So in plain language:

> If you cannot draw a line between two events, then we are unable to establish a causal relation between those events

Or to use fancier, academic language:

> Causality requires graph reachability in spacetime

So in summary, when we reason about Lamport Clock values, we understand that for events `A` and `B`:

If we know that `A -> B`, then we can be sure that `LC(A) < LC(B)`

In other words:

> Lamport Clocks are consistent with causality

However, simply knowing that `LC(A) < LC(B)` does not prove `A -> B`

This is because

> Lamport Clocks do not characterise (or establish) causality

The above example is derived from a paper by Reinhard Schwarz and Friedemann Mattern called [Detecting Causal Relationships in Distributed Computations: In Search of the Holy Grail](vs.inf.ethz.ch/publ/papers/holygrail.pdf)


### So What Are Lamport Clocks Good For?

Even though a Lamport Clock cannot characterise causality, they are still useful.

We have the logical implication: `if A-> B then LC(A) < LB(B)`

All logical implication are constructed from a premise (`A-> B`) stated in the form of a question, and a conclusion (`LC(A) < LB(B)`) that can be reached if the question is true.

For any logical implication, we can also take its contra-positive

Thus, if `P => Q` then the contra-positive states that `¬Q => ¬P`

So in the case of the "happens before" relation

`if A-> B then LC(A) < LB(B)`

The contra-positive states

`if ¬(LC(A) < LB(B)) then ¬(A->B)`

The contra-positive states that if the Lamport Clock of `A` is not less than the Lamport Clock of `B`, then `A` cannot have happened before `B`.  This turns out to be really valuable in debugging because with complete certainty, we can rule out that one event did not happen before another, and therefore cannot have contributed to the problem.


## Summary

Lamport Clocks are good for certain aspects of determining causality, but they do leave us with indeterminate situations.  This is because a Lamport Clock is consistent with causality, but does not characterise it.

In order to remove this indeterminaccy, we need a different type of clock, called a Vector Clock.

