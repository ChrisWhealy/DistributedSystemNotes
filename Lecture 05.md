# Distributed Systems Lecture 5

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 8<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=zuxA6f-XIAc)

| Previous | Next
|---|---
| [Lecture 4](./Lecture%2004.md) | [Lecture 6](./Lecture%2006.md)



## Recap of Lamport Clocks

Lamport Clocks are consistent with causality, but do not characterise (or establish) it.

If `A -> B` then this implies that `LC(A) < LC(B)`

However, this implication is not reversable:

If `LC(A) < LC(B)` then it does ***not*** imply `A -> B`


## Vector Clocks

Invented independently by Friedemann Mattern and Colin Fidge.  Both men wrote papers about this subject in 1988.

> Addendum
> 
> Subsequent investigation by Lindsey Kuper has uncovered that without using the specific name *"vector clock"*, the concept was already being used by Rivka Ladin and Barbara Liskov in 1986.
> See [lecture 23](https://github.com/ChrisWhealy/DistributedSystemNotes/blob/master/Lecture%2023.md#ladin--liskovs-paper-highly-available-distributed-services-and-fault-tolerant-distributed-garbage-collection) for details.

For Vector Clocks however, this relation ***is*** reversable:

`A -> B <==> VC(A) < VC(B)`

A Lamport Clock is just a single integer, but a Vector Clock is a sequence of integers.
(The word *"vector"* is being used here in the programming sense, not in the sense of a magnitude and direction)

### Implementing a Vector Clock

There are two pre-conditions that must be fulfilled before you can implement a Vector Clock:

1. You must know upfront how many processes make up your system
1. All the processes must agree the order in which the clock values will occur within the vector 

In this case, we know that we have three processes `Alice`, `Bob` and `Carol`, and that the order of values in the vector will be sorted alphabetically by process name.

So, each process will create its own copy of the vector clock with an initial value of `[0,0,0]` for `Alice`, `Bob` and `Carol` respectively.

> ***IMPLEMENTATION DETAIL***  
> When implementing a vector clock, the above two constraints can be relaxed.
> 
> Rather than implementing a vector clock as a simple list of integers, it is useful first to create data type for a Lamport Clock which could be as simple as just the process name and a clock value.
> 
> A minimal Rust implementation might look something like this:
>     
> ```rust
> pub struct LamportClock {
>   pub process_name: String,
>   pub clock_value: u64
> }
> ```
>     
> A minimal Vector Clock is then simply an array of Lamport Clocks.
>     
> ```rust
> pub struct VectorClock {
>   pub entries: Vec<LamportClock>
> }
> ```
> 
> Now with suitable coding to manage vector clocks, process names can occur in any order, or even be missing.

The Vector Clock is then managed by applying the following rules:

1. Every process maintains a vector of integers initialised to `0` - one for each process with which we wish to communicate
    
1. On every event, a process increments its own position in the vector clock: this also includes internal events that do not cause messages to be sent or received

1. When sending a message, each process includes the current state of its clock as metadata with the message payload

1. When receiving a message, each process updates its own position in the vector clock using the rule `max(VC(self),VC(msg))`

If, at some point in time, the state of the vector clock in process `Alice` becomes `[17,0,0]`, then this means that as far as `Alice` is concerned, it has recorded 17 events, whereas it thinks that processes `Bob` and `Carol` have not recorded any events yet.

### Calculating the `max` of Two Vector Clocks

But how do we take the `max` of two vectors?

The notion of `max` we're going to use here is a per-element comparison (a.k.a. a pointwise maximum)

For example, if my VC is `[1,12,4]` and I receive the VC `[7,0,2]`, the pointwise maximum would be `[7,12,4]`

### Applying the `<` Operator on Two Vectors

What does `<` mean in the context of two vectors?

It means that when a pointwise comparison is made of the values in `VC(A)` and `VC(B)`, two things must be true:

* At least one value in `VC(A)` is less than the corresponding value in `VC(B)`, and
* No value in `VC(A)` is greater than the corresponding value in `VC(B)`

> ***IMPORTANT***   
> For two vector clocks to be comparable, they must refer to exactly the same set of clock values, and depending on your implemention, those values may also have to be listed in the same order.

This comparison can be performed using the `≤` operator as long as we first reject the case where `VC(A) == VC(B)`.
To put that more algorithmically, the following must be true (and here we assume that a vector clock is a simple list of integers):

<pre>
   VC(A) !== VC(B)
&& VC(A).length == VC(B).length
&& for each process at position i, VC(A)<sub>i</sub> ≤ VC(B)<sub>i</sub>
</pre>

### But Is This Comparison Enough to Characterise Causality?

Consider two Vector Clocks `VC(A) = [2,2,0]` and `VC(B) = [1,2,3]`

Is `VC(A) < VC(B)`?

So, taking a pointwise comparison of each element gives:

| Index | <code>VC(A)<sub>i</sub> ≤ VC(B)<sub>i</sub></code> | Outcome
|---|---|---
| `0` | `2 ≤ 1` | `false`
| `1` | `2 ≤ 2` | `true`  
| `2` | `0 ≤ 3` | `true`  

The overall result is then calculated by `AND`ing all the outcomes together:

`false && true && true = false`

So, we can conclude that `VC(A)` is ***not*** less than `VC(B)`.

Ok, let’s do this comparison the other way around.

Is `VC(B) < VC(A)`?

Again, we perform a pointwise comparison of each element gives:

| Index | <code>VC(B)<sub>i</sub> ≤ VC(A)<sub>i</sub></code> | Outcome
|---|---|---
| `0` | `1 ≤ 2` | `true`
| `1` | `2 ≤ 2` | `true`  
| `2` | `3 ≤ 0` | `false`  

The overall result is still `false` because `true && true && false = false`

Since `VC(B)` is ***not*** less than `VC(A)` and `VC(A)` is ***not*** less than `VC(B)`, we are left in an indeterminate state.
All we can say about the events represented by these two vector clocks is that they are concurrent, independent or causally unrelated (these three terms are synonyms).

I.E. `A || B`

## Worked Example

Let's see how the vector clocks in three processes (`Alice`, `Bob` and `Carol`) are processed as messages are sent and received:

![Vector Clocks example 1](./img/L5%20VC%20Clocks%201.png)
![Vector Clocks example 2](./img/L5%20VC%20Clocks%202.png)
![Vector Clocks example 3](./img/L5%20VC%20Clocks%203.png)
![Vector Clocks example 4](./img/L5%20VC%20Clocks%204.png)
![Vector Clocks example 5](./img/L5%20VC%20Clocks%205.png)
![Vector Clocks example 6](./img/L5%20VC%20Clocks%206.png)
![Vector Clocks example 7](./img/L5%20VC%20Clocks%207.png)

After these messages have been sent, we can see the history of how the vector clocks in each process changed over time.

![Vector Clocks example 8](./img/L5%20VC%20Clocks%208.png)

## Determining the Causal History of an Event

If we choose a particular event `A`, let's determine the events in `A`'s causal history.

![Causal History 1](./img/L5%20Causal%20History%201.png)

`A` was an event that took place within process `Bob` and by looking back at `Bob`'s other events, we can see one sequence of events:

![Causal History 2](./img/L5%20Causal%20History%202.png)

Also, by following the messages that led up to event `A`, we can see another sequence of events:

![Causal History 3](./img/L5%20Causal%20History%203.png)

What is common here is that: 

1. Event `A` has ***graph reachability moving forwards in spacetime*** from all the events in its causal history.  
    That is, without lifting the pen from the paper or going backwards in time, we can connect any event in `A`'s past with `A`.
1. Working backwards from `A`, we can see that all the vector clock values in its causal history satisfy the ***happens before*** relation: that is, all preceding vector clock values are less than `A`'s vector clock value.

In addition to this, by looking at events that come after `A` (in other words, `A` is in the causal history of some future event), we can see that all such vector clock values are larger than `A`'s.

### Are the Vector Clocks of All Events Comparable?

Consider events `A` and `B`.
Can their vector clock values be related using the ***happens before*** relation?

![Causal History 4](./img/L5%20Causal%20History%204.png)

In order for this relation to be satisfied, the vector clock of one event must be less than the vector clock of the other event (in a pointwise sense).
But in this case, this is clearly not true:

```
VC(A) = [2,4,1]
VC(B) = [0,3,2]

[2,4,1] < [0,3,2] = false
[0,3,2] < [2,4,1] = false
```

Neither vector clock is larger or smaller than the other; therefore, all we can say about these two events is that they are independent, concurrent or causally unrelated, or `A || B`.

This does however mean that we can easily tell a computer how to determine if two events are causally related.
All we have to do is compare the vector clock values of these two events.
If we can determine that one is less than the other, then we know for certain that the event with the smaller vector clock value occurred in the causal history of the event with the larger vector clock value.

If, on the other hand, the ***less than*** relation cannot be satisfied, then we can be certain that the two events are causally unrelated.

## Protocols

The non-rigorous definition of a protocol is that it is an agreed upon set of rules that computers use for communicating with each other.

Let's take a simple example:

One process sends the message `Hi, how are you?` to another process.
According to our simple protocol, when a process receives this specific message, it is required to respond with `Good, thanks`

However, our simplistic protocol says nothing about which process is to send the first message, or what should happen after the `Good, thanks` message has been received.

The following two diagrams are both valid runs of our protocol:

![Protocol 1](./img/L5%20Protocol%201.png)

![Protocol 2](./img/L5%20Protocol%202.png)

What about this?

![Protocol 3](./img/L5%20Protocol%203.png)

Nope, this is not allowed because our protocol states that the message `Good, thanks` should only be sent in response to the receipt of message `Hi, how are you?`.
Therefore, sending such an unsolicited message constitutes a protocol violation.

### So How Long Did the Message Exchange Take?

What about this - is this a protocol violation?

![Protocol 4](./img/L5%20Protocol%204.png)

Hmmm, it's hard to tell.
Maybe the protocol is running correctly and we're simply looking at a particular point in time that does not give us the full story.

The point here is that a Lamport Diagram can only represent logical time - that is, it describes the order in which a sequence of events occurred, but it cannot give us any idea about how much real-world time elapsed between those events.

But considering that a Logical Clock is only concerned with the ordering of events, it is not surprising that the following two event sequences appear to be identical:

![Protocol 5](./img/L5%20Protocol%205.png)

### Complete Protocol Representation?

***Q:***&nbsp;&nbsp; Is it possible to use a Lamport Diagram to give us a complete representation of all the possible message exchanges in a given protocol?  
***A:***&nbsp;&nbsp; No!

It turns out that there are infinitely many different Lamport Diagrams that all represent valid runs of a protocol.

The point here is that a Lamport Diagram is good for representing a ***specific*** run of a protocol, but it cannot represent ***all possible*** runs of that protocol, because there will be infinitely many.

Lamport Diagrams are also good for representing protocol violations - for instance, in the diagram above, `Alice` sent the message `Good, thanks` to `Bob` without `Bob` first sending the message `Hi, how are you?`.

### Correctness Properties and Their Violation

When discussing properties of a system that we want to be true, one way to talk about the correctness of such properties is to draw a diagram that represents its violation.

For instance, consider the order in which messages are sent and received.

### Important Terminology

In distributed systems, messages are not only sent and received, but they are also ***delivered***.

Hmmmm, it sounds like there is some highly specific meaning attached to the word ***deliver***...  Yes, there is.

***Sending*** and ***receiving*** can be understood quite intuitively, but in the context of distributed systems, the concept of ***message delivery*** has a highly specific meaning:

* ***Sending a Message***  
    The explicit act of causing a message to be transmitted (typically over a network).  
    The sending process has complete control over when or even if a message is sent; therefore, sending a message is entirely ***active***.
* ***Receiving a Message***   
    The act of capturing a message upon arrival.  
    The receiving process has no control over when or even if a message arrives &mdash; they just show up randomly... or not.  Therefore, from the perspective of the receiving process, receiving a message is entirely ***passive***.
* ***Delivering a Message***  
    The term ***Delivery*** exists to distinguish the passive act of receiving a message from the active choice to start processing the contents of that message. You can't decide when you're going to receive a message, but you can decide when and if to process its contents; therefore, although receiving a message is passive, the choice to deliver a message is entirely ***active***.   
    For example, by placing a received message into a queue, we can delay acting upon the contents of that message until some later point in time.

### FIFO (or Ordered) Delivery

If a process sends message `M2` after message `M1`, the receiving process must deliver `M1` first, followed by `M2`.

We can represent a protocol violation such as ***FIFO anomaly*** using the following diagram. 

![FIFO Anomaly](./img/L5%20FIFO%20Anomaly.png)

This is an example of where a diagram provides a very useful way to represent the violation of some correctness property of a protocol.


---

| Previous | Next
|---|---
| [Lecture 4](./Lecture%2004.md) | [Lecture 6](./Lecture%2006.md)


