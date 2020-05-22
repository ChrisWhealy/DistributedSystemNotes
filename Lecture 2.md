# Distributed Systems Lecture 2

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 1st, 2020 via [YouTube](https://www.youtube.com/watch?v=G0wpsacaYpE)

| Previous | Next
|---|---
| | [Lecture 3](./Lecture%203.md)

## What is a Distributed System?

Leslie Lamport gives the rather comical definition that:

> *"A distributed system is one in which I can't get my work done because a computer I've never heard of has crashed"*

Although he was joking, this definition captures a very important aspect of a distributed system in that it is one defined by some type of failure.

Martin Kleppmann's definition of a distributed system is somewhat more serious (he's the author of a book called [Designing Data Intensive Systems](https://www.amazon.co.uk/Designing-Data-Intensive-Applications-Reliable-Maintainable-ebook/dp/B06XPJML5D/ref=sr_1_1).

> A distributed system runs on several nodes (computers) and is characterised by partial failure

However, other definitions of a distributed system include ideas such as:

- Systems where work is distributed fairly between nodes, or
- Where multiple computers "behave as one"

These last definitions, however, are all rather too optimistic because they do not account for any real-life difficulties - one of which is mentioned in Martin Kleppman's definition.

## What is Partial Failure?

Partial failure is where some component of a computation fails, but that failure is not necessarily fatal.  For instance, one machine in a cluster of 1000 could fail without there being any significant impact on the overall "system".  In other words, the presence of this type of partial failure is non-fatal to the operation of your system as a whole.


## "Cloud Computing" vs. "High Performance Computing (HPC)"


The problem with HPC is that it treats partial failure as total failure.  Consequently, HPC must rely on techniques such as check-pointing.  This is where the progress of the calculation is saved at regular intervals so that in the event of a failure, the computation can be continued from the last check point without needing start over from the beginning.


However, in Cloud Computing, it is expected that various parts of the system will fail.  Consequently, this type of behaviour is a fundamental part of the software design.

![Ben Franklin](./img/Ben%20Franklin.jpg)

Failure can occur in many ways.  For instance:

- Network partition
- Hardware failure
- Software failure

### Determining the Cause of Failure

In a minimal cluster of two computers `M1` and `M2`, `M1` wants to ask `M2` for the value of a particular variable:

- `M1` sends message to `M2` saying *"What's the value of `x`?"*
- `M2` should then respond with another message saying *"`x=5`"*

![Message 1](./img/L2%20Message%201.png)

But in in this very simple scenario, the number of possibilities for failure is still very high:

- `M1`'s message might get lost due to network failure
- `M1`'s message is delivered very slowly due to unexpectedly high network latency
- `M2` could be down
- `M2` might crash immediately after reporting to `M1` that it is alive, but before receiving `M1`'s message
- `M2` might crash as a result of trying to respond to `M1`'s message
- `M2` might refuse to respond to the question due to some type of authentication or authorisation failure
- `M2` responds correctly, but the message never gets back to `M1`
- Some random external event such as a cosmic ray flips a bit in the message thus corrupting it (Byzantine error)

And on and on...

Although the underlying causes are very different, as far as `M1` is concerned, they all look the same.  As far as `M1` can tell, its question was simply never answered.

In fact, it is impossible to determine the cause of such a failure without first having global knowledge of the entire system.

### Timeouts

It is often assumed that `M1` must wait for some predefined timeout period, after which it should assume failure if it does not receive an answer.  But the problem is that network latency is indeterminate; therefore, waiting for an arbitrary timeout period is not a good strategy for determining failure.

For example, instead of `M1` asking *"What is the value of `x`"*, it asks `M2` to *"Add 1 to `x`"*, then `M1` is only going to know that its request was successful when it receives some sort of acknowledgement back from `M2`.

If `M1` does not receive a response within its timeout period, what should it conclude about the success or failure of that request?

We can see that without any additional information, it is not correct for `M1` to assume failure.  All `M1` can say with any certainty is that from its point of view, the state of variable `x` in `M2` is indeterminate.

### Realistic Timeout Values?

If the maximum network delay for message transmission is `D` seconds and the maximum time a machine spends processing a request is `R`, then the upper bound of the message handling timeout should be `2D + R` (two network journeys (send and receive) plus the remote machine's processing time).  This would rule out the uncertainty for timeouts in a slow network, but still leave us completely unable to reason about any other types of failure.

In distributed systems, we must deal not only with the problems of "Partial failure", but also the problem of "Unbounded Latency" (the definition given by Peter Alvaro - one of Lindsey Kuper's colleagues)

Given they're so hard to debug, why would you want to use a distributed system?  Well, largely because we have no choice.  All manner of external factors can force us to distribute either a computation or data storage (or both) across multiple machines&hellip;

- Too much data to fit on a single machine
- Need a faster response time, so we have to throw more processing power at the problem
- Scalability (need to handle more data, more users, or simply need more CPUs)
- Fault Tolerance (redundancy)


## Time and How We Measure it

Computers use clocks to identify specific points in time (both past and present).  For example:

- This class starts at 09:00
- This item in the cache will expires tomorrow at 4:26 PM
- This log event happened at yesterday at 11:31:34.352

Computers also use clocks for reasoning about time intervals:

- This class is 65 minutes long
- This access code will expire in 30 seconds time
- This user spent 4 minutes 38 seconds on our website


### Time-of-Day Clocks

Computers have two type of clock, time-of-day clocks and monotonic clocks.

- Time-of-day clocks are typically synchronised across different machines using [NTP](http://www.ntp.org/)
- Time-of-day clocks are ***bad*** for measuring intervals between events:
    -  The clocks on the different machines may not be synchronised correctly and thus will disagree on the exact size of a time interval
    -  There are several cases where the time-of-day clock can jump (usually backwards):
        - Daylight saving time just started
        - A leap second happens
        - NTP resets the machine's clock to an earlier value
- Time-of-day clocks are OK for timestamping particular events, but it depends on the degree of accuracy needed, which in turn is due to clock synchronisation being only so-good

There is a general principle in timekeeping: the more accurately you need to measure time, the harder that task becomes.  If you now try to get two or more computers to agree on what the time is, then the difficulty is only compounded.

### Monotonic clocks

A value is said to be *"monotonic"* if it only ever changes in one direction.  So, a monotonic clock is one that will ***never*** jump backwards; its counter is guaranteed only to get bigger.

The properties of a monotonic clock are:

- Its value only ever goes forwards E.G. microseconds since system boot
- Uses an arbitrary value that is meaningless outside the scope of the machine on which it exists; therefore, it makes no sense to compare monotonic clock values between different machines, and they are useless as timestamps
- Good for duration or interval measurement within the context of a single machine

### What's the Time?

So how do we mark points in time that are valid across multiple machines?  To answer this question to any high degree of precision turns out to be very tricky.

Time of day and monotonic clocks are both physical clocks (I.E. they are concerned with quantifying a time interval between now and some fixed reference point in the past such as Jan 1st, 1970 or when the machine was started)

### Which Came First?

In a distributed system however, we work with a very different notion of what a clock is.   In distributed systems, we work with the concept of a ***Logical Clock***.

Counter-intuitively, a logical clock measures neither the time of day nor the elapsed interval between two events; instead, they measure nothing more than the ordering of events.

At first, this concept is not easy to grasp because it goes against our firmly established notion of what a clock is (or ought to be).

However, the purpose of a logical clock is to help us answer the question ***"Which event happened first?"***.

This is very important to know for several reasons:

* Communication is unreliable  
    This means that the length of time taken for message transmission is unpredictable (a phenomenon known as *"unbounded latency"*)
* Unpredictable changes of state  
    E.G. the last time you asked machine `M2` what the value of `x` was, it said `x=5`; but in the meantime, some other machine has changed `x` to `6` and hasn't told you&hellip;

In these situations, it is vital to know the order in which events occurred.

### Event Ordering

Let's says we have two database events `A` and `B` and we know that `A` happened before `B`.  This relationship is denoted by the syntax:

`A -> B`

And is pronounced *"`A` happens before `B`"*

What does this tell us about causality?

All we can really say is that event `A` ***might*** have been the cause of event `B`, but we cannot be certain about this.  All we can say with absolute certainty is that since `B` happened after `A`, it is ***impossible*** for `B` to have been the cause of `A`.

Questions about causality are very important in distributed systems because they help us solve problems such as debugging.

If `M1` thinks `x=1`, but `M2` thinks `x=5`, then by knowing when the value of `x` changed in each machine, we can rule out those events that did ***not*** contribute to the problem.  This greatly helps in narrowing down what ***is*** the cause of the problem.

This determinism is also useful when designing systems in which users see a sequence of events.  You know that if event `A` happened before event `B`, then you can ensure that your system will ***never*** display event `B` to the user before they have first seen event `A`.

---

| Previous | Next
|---|---
| | [Lecture 3](./Lecture%203.md)


