# Distributed Systems Lecture 16

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 6<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=h8wGZVOh43c)

| Previous | Next
|---|---
| [Lecture 15](./Lecture%2015.md) | [Lecture 17](./Lecture%2017.md)


## Course Admin

...snip...

### Review of Question in the Midterm Exam

Lots of people struggled with this question in the midterm exam, and since it is a fundamental concept, if you don't understand this, you'll struggle later with other concepts based on it.

The question was this: *"In a distributed system, what should a physical clock be used for and what should it not be used for?"*

So, what are physical clocks good for in a distributed system?

We spoke of two types of physical clock:

* A time-of-day clock
* A monotonic clock

These two types of physical clock stand in contrast to a logical clock.

| | Time-of-day Clock | Monotonic Clock
|---|---|---|
| Marking points in time | ![Neutral emoji](./img/emoji_neutral.png)<br>OK for processes running on a single system, but not much good in distributed systems because there can be no shared clock. | ![Sad emoji](./img/emoji_sad.png)<br>Completely useless because they are simply counters that increment from some ["unspecified point in the past"](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap04.html#tag_04_16) which is unique to each machine (E.G. Nanoseconds since system start).<br>Monotonic clock values from different machines cannot be meaningfully compared.
| Durations/Intervals | ![Sad emoji](./img/emoji_sad.png)<br>The problem here is that time-of-day clocks can jump around due to daylight saving time and leap seconds.| ![Smiley emoji](./img/emoji_smiley.png)<br>On a single machine, these are great because they only go forward from some "unspecified point in the past".

***Q:***&nbsp;&nbsp; So, what are physical clocks good for in distributed systems?  
***A:***&nbsp;&nbsp; Timeouts!

Timeouts (measured using a monotonic clock) are used in a wide variety of places for failure detection.  E.G. A Paxos proposer uses a timeout when waiting for responses to a new proposal number.

***Q:***&nbsp;&nbsp; So, what are physical clocks ***not*** good for in distributed systems?  
***A:***&nbsp;&nbsp; Measuring the order of events

Why?  Because if you try to determine the order in which a sequence of events have occurred by marking each event's point in time, we will run into problems when we try to compare time values recorded on different machines.

If we use time-of-day clocks, then these will only be comparable in a very course sense because they are prone to jumping around due to factors such as daylight-saving time and leap seconds.

If we try to use monotonic clock values, then things get even worse because they are simply counters from some *"unspecified point in the past"*; thus, it is meaningless to compare values coming from different machines.

So physical clocks are simply not suitable for measuring the order of events that occurred on different machines.

## Paxos: Nontermination

All consensus algorithms try to satisfy the following three properties:

* Termination
* Validity / Integrity
* Agreement

We have already stated that for a consensus algorithm running in a distributed system where failures are possible, it is impossible to implement all three properties.  This is known as the [FLP Result](./papers/FLP.pdf).

Therefore, all consensus algorithms must compromise on one these properties, and Paxos compromises on the termination property &mdash; which risks cases where non-termination could occur.

So, what kind of situation would lead to non-termination?

Simply put, the Paxos algorithm will not terminate in cases where multiple proposers contend with each other for the proposed value.

Consider this sequence of events:

![Paxos Nontermination 1](./img/L16%20Paxos%20Nontermination%201.png)

* Proposer <code>P<sub>1</sub></code> sends out `prepare(5)` messages to acceptors <code>A<sub>1</sub></code> and <code>A<sub>2</sub></code>
* Neither acceptor has seen this proposal number yet, so they both respond with `promise(5)`
* But quite independently of this, proposer <code>P<sub>2</sub></code> sends out its `prepare(6)` messages to acceptors <code>A<sub>2</sub></code> and <code>A<sub>3</sub></code>
* Even though acceptor <code>A<sub>2</sub></code> has already promised to ignore proposal numbers less than `5`, it has not yet accepted any values at this proposal number, so it's not a problem if it now promises to accept this new, higher proposal number
* Acceptor <code>A<sub>3</sub></code> is receiving its first proposal number, so it happily promises to ignore proposal numbers lower than `6`.

Unaware of what the other acceptors have just agreed to, proposer <code>P<sub>1</sub></code> now sends out its `accept(5,"foo")` message thinking that it already has agreement from the majority of acceptors.

![Paxos Nontermination 2](./img/L16%20Paxos%20Nontermination%202.png)

* Acceptor <code>A<sub>1</sub></code> is still on proposal number `5`, so it sends back an `accepted(5,"foo")` message as expected
* But due to proposer <code>P<sub>2</sub></code>'s earlier `prepare(6)` message (that proposer <code>P<sub>1</sub></code> knows nothing about), acceptor <code>A<sub>2</sub></code> has now promised to ignore messages with a proposal number lower than `6`.
* So, proposer <code>P<sub>1</sub></code> sits there waiting for a response.  It fails to receive responses from the majority of acceptors, and eventually times out.
* Not being one to give up, <code>P<sub>1</sub></code> sends out another `prepare` message, this time for the higher proposal number `7`.
* Just as before, acceptors <code>A<sub>1</sub></code> and <code>A<sub>2</sub></code> are happy to oblige and respond with `promise(7)` messages.

Now, unaware of all the collusion going on between <code>P<sub>1</sub></code> and the acceptors, <code>P<sub>2</sub></code> sends out its `accept(6,"bar")` message.

![Paxos Nontermination 3](./img/L16%20Paxos%20Nontermination%203.png)

* Acceptor <code>A<sub>3</sub></code> happily responds with `accepted(6,"bar")`
* But since acceptor <code>A<sub>2</sub></code> has already promised to ignore messages with a proposal number lower than `7`, it simply ignores <code>P<sub>2</sub></code>'s `accept` message
* <code>P<sub>2</sub></code> hangs around waiting for a majority of acceptors to respond, but this never happens, so it times out
* Not being one to give up, <code>P<sub>2</sub></code> sends out another `prepare` message, this time for the higher proposal number `8`.


And on and on we can go here - with each proposer continually trying to outbid the others.  Consensus will never be reached.

This is known as the ***"Duelling Proposers Problem"***.

In this case, we have used three acceptors, but actually only two would have been sufficient to demonstrate non-termination.

### So, Why Not Just Have One Proposer?

All of this confusion seems to be created by the fact that we have multiple proposers all trying to outbid each other.  Couldn't we solve this problem simply by restricting the number of proposers to one?

Well, let's see what would happen if we tried to enforce that there is exactly one proposer:

* When considering fault tolerance, if we ensure that the proposer always writes its data to persistent storage, then even if it dies, we can nominate another process to act as the new proposer, and then we'll be able to carry on as before.
* But hang on, how do we decide which node will take over?
* Oh, that's a leader election problem...
* And to solve a leader election problem we need consensus
* And if we use Paxos to elect a new leader, that algorithm might never terminate

And we're back to where we started...

So, we cannot insist on there being only one proposer because our consensus algorithm depends on a consensus algorithm.  For instance, if we relied on Paxos to determine a new leader, then we're depending on an algorithm that might never terminate.

Some consensus algorithms go through a phase in which they elect a leader (which requires consensus) and then that leader becomes the sole proposer.  This does not eliminate the possibility of the *Duelling Proposers Problem* (because it could still occur during leader election phase); however, it confines non-termination to the leader election phase and removes it from the value proposal phase.

This is strategy for reducing risk, not removing it.

## Would It Ever Make Sense to Compromise on a Different Property?

So far, we have seen that the Paxos algorithm values *agreement* and *validity* over termination; therefore, it sees non-termination as an acceptable risk in the quest to achieve agreement and validation.

But is there another way we could work here?

### What Would Happen if we Compromised on Agreement?

Leader election is one situation in which it is more important for the algorithm to terminate than it is for everyone to agree.

So now we could elect the proposer in the Paxos algorithm using a different leader election algorithm that risks producing multiple values, but we know will terminate.  If this algorithm elects a single proposer most of the time, then that's great; however, if it elects multiple proposers, then this is also OK because we know that Paxos can work with multiple proposers.

So, this leads us nicely into the next topic - Multi-Paxos.

## Multi-Paxos

All the runs of the Paxos algorithm that we've looked at so far are concerned with deciding on a single value.  If we want to decide on a sequence of values, then the Paxos algorithm must be rerun.

As it turns out, agreeing on a sequence of values is a widespread problem; for example, in [lecture 14](./Lecture%2014.md#when-do-you-need-consensus), we looked at a list of problems that all require consensus, and one of these was the problem of Totally-Ordered Broadcast

> Remember that in Totally-Ordered Broadcast, we need to ensure that a set of processes all deliver a set of messages in the ***same order***.  Therefore, consensus must be reached on which message should be delivered first, which second and which third etc.
> 
> This turns out to require us to make the same decision over and over for each message in the queue.

The problem here is that in the best case, in order for Paxos to decide on a single value, a minimum of two round trips are needed between the proposer and the majority of acceptors.  In the case that we have three acceptors, this will require a total of 8 messages to be exchanged between the proposer and at least two of the acceptors.

![Minimum Number of Message Exchanges Needed by Paxos](./img/L16%20Paxos%20Minimum%20Msg%20Exchange.png)

In order for this minimum to be achieved, it is assumed that:

* All processes are correct (I.E. they don't crash or exhibit some weird Byzantine behaviour)
* Only one proposer is involved

However, what's the downside here?  Having to repeat this message exchange sequence over and over again is not very efficient!

### Paxos Algorithm Phases

A full run of the Paxos algorithm is divided into two distinct phases:

* The prepare/promise phase, and
* The accept/accepted phase

![Paxos Algorithm Phases](./img/L16%20Paxos%20Phases.png)

However, we can apply an optimisation here.

After a single run of the Paxos algorithm has completed, the proposer knows that the majority of acceptors have agreed to its last proposal number; therefore, if we require consensus on another value, we can skip the prepare/promise phase and simply repeat the accept/accepted phase for the next value.

For Multi-Paxos to work successfully however, we need to make two assumptions:

1. The proposal number does not change (hopefully because there is only one proposer)
1. The proposer process does not crash

Since we want consensus on a sequence of values, we can arbitrarily add our own additional sequence number into the `accept` messages to keep track of this value.


### Multi-Paxos in Action

So, let's say we want consensus on the sequence of values `"foo"` and `"bar"`.  We still need to go through the prepare/promise phase, but we only need to do that once.  After that, we can simply repeat the accept/accepted phases:

![Multi-Paxos](./img/L16%20MultiPaxos.png)

Notice that the `accept` and `accepted` messages now contain an additional sequence number after the value:

* `prepare(5)` is answered with `promise(5)` as normal
* <code>P<sub>1</sub></code> now sends an `accept(n, val, seq)` message where:
    * `n` is the promised proposal number,
    * `val` is the value upon which consensus is being sought, and
    * `seq` is some arbitrary sequence number


Ok, but what happens if we do have a second proposer who starts injecting their own proposal numbers?

In this case, we will not be able to repeat the second phase because some of the acceptors will be ignoring <code>P<sub>1</sub></code>'s `accept` messages.  Now, <code>P<sub>1</sub></code> will simply time out and start the prepare/promise phase again.  In other words, nothing breaks and Multi-Paxos gracefully degenerates back to regular Paxos &mdash; however, we do not expect this situation to happen very often.

Using this approach, Totally-Ordered Delivery can be achieved by a set of processes using the (Multi-)Paxos consensus algorithm between them to agree on message delivery order.

## But Is This the Whole Story?

Here's another type of consensus algorithm:

* We have one acceptor process
* We have multiple proposer processes
* Each proposer sends their value to the acceptor who somehow decides which value to accept
* Whatever value is returned from the acceptor is now the *"agreed"* value.

Boom! All the required properties have been fulfilled!

Why can't we just do this?

Well how about fault tolerance?  What if our single acceptor process crashes?

### Tolerating Crash Faults

As we've mentioned before, the [FLP result](./papers/FLP.pdf) demonstrates that if we need to be able to withstand crashes, then it is impossible to have a consensus algorithm that exhibits all three properties of agreement, validity and termination.

Consensus algorithms tend to be designed in a very defensive way in order to make them as robust as possible in the event of processes crashing &mdash; which is part of the reason for all the complexity in the Paxos algorithm.

#### Acceptor Failure

In the examples we've worked with so far, we have always had three acceptors.  So how many of these acceptors could crash without bringing Paxos down?

* If we start with three acceptors, then two is a majority
* If one acceptor crashes, we are still left with a majority of the original three
* Therefore, Paxos still works

Similarly, if we have five acceptors, then by the above reasoning, we can tolerate up to two acceptors crashing.

In general then, if `f` is the maximum number of failed acceptors we can tolerate, then we must start with a total of `2f + 1` acceptors.

#### Proposer Failure

Again, if we start with the idea that `f` is now the number proposer failures we are able to tolerate, then we must decide how many proposers our system must have when it starts.

We know from the above discussion, that Paxos actually works very well if there is only one proposer, so 1 is the minimum we can work with.  If we can tolerate `f` failures, then it is clear that our system should start with at least `f + 1` proposers.

This is the degree to which Paxos can tolerate crash faults.

### Tolerating Omission Faults

How well does Paxos tolerate omission faults?

What happens if a proposer sends a `prepare` message to all the acceptors, but for some reason that message never arrives at some of those acceptors?

Well, the proposer only needs to hear back from a majority of acceptors; so, if a `prepare` message does not arrive at one out of three acceptors, then this will not create a problem.  The proposer will still hear back from two out of three which is a majority.

Ok, let's ramp up the severity &mdash; what if the proposer sends a `prepare` message to all three acceptors, and all three messages got lost?

Again, other than slowing things down, nothing bad would happen because after waiting for its time out period, the proposer would simply try again with a new proposal number.  At this point, we might run into a non-termination problem, but that is not an omission fault.

So Paxos does OK in the case of omission faults - it might not terminate, but as we've seen with the *"Duelling Proposers Problem"* shown above, we don't need to experience message loss in order for non-termination to occur.

## Other Consensus Protocols

Here we will mention a few other consensus protocols.  We will not look into them in any detail, but since we have had a detailed look at Paxos, you now have a good foundation from which to study these algorithms on their own.

These consensus algorithms all have a couple of common features:

1. They are designed to achieve consensus on a sequence of values, not just one - which makes them more like Multi-Paxos than basic Paxos
1. They all include leader-election as a fundamental part of the protocol

These consensus protocols are:

* ***Viewstamped Replication (VSR)***  
    Developed by Brian Oki and Barbara Liscov (1988) and documented in this [paper](./papers/VSR.pdf).  
    An explanation of this protocol by Bruno Bonacci is available on YouTube [here](https://www.youtube.com/watch?v=1EzNa-zAYS8)

* ***ZAB (Zookeeper Atomic Broadcast)***  
    Developed at Yahoo in the late 2000's as part of their Open Source Zookeeper system often used for configuration management.  
    A paper by André Medeiros on the theory and practice of ZAB is available [here](./papers/ZAB.pdf).

* ***Raft***  
    Developed by Diego Ongaro and John Ousterhout in 2014.  The whole point of Raft is understandability.  
    It is very well explained on their website <https://raft.github.io>.

Also, [this paper](./papers/Paxos%20vs%20VSR%20vs%20ZAB.pdf) provides a useful comparison of the Paxos, VSR and ZAB algorithms.


## Active and Passive Replication

The following statement is confused:

*"Primary Backup replication is active replication"*

To remove this confusion, we should first define exactly what these terms mean.

Let's say we're doing Primary Backup on a replicated back account.  This is the situation we looked at in [lecture 12](https://github.com/ChrisWhealy/DistributedSystemNotes/blob/master/Lecture%2012.md#primary-backup-replication).

We receive an instruction to deposit \$50 into an account that already has a balance of \$20.  But there are two ways we could implement this deposit operation.

### Active Replication

We could apply the deposit instruction to the primary `P` and then broadcast that instruction to both backups.  These backups then apply that instruction to their own replicas of the account.  

When the `ack`s from all the backups have been received by the primary, the primary delivers the message to itself and finally sends an `ack` to the client.
    
![Primary Backup Replication 1](./img/L16%20Primary%20Backup%201.png)

Now processes `P`, <code>B<sub>1</sub></code> and <code>B<sub>2</sub></code> hold identical state because they all applied the same instruction to the same initial account balance

This is active replication because the operation has been executed on every replica where we want it to take effect.

### Passive Replication

Alternatively, the primary `P` could execute the deposit instruction, but not commit it yet.  

Now that `P` knows what the new balance is, instead of sending the deposit instruction, it simply sends the new account balance to the backups.

![Primary Backup Replication 2](./img/L16%20Primary%20Backup%202.png)

The backups simply apply store the new state of the account and send back their `ack` messages.  Then, as before, the primary sends an `ack` to the client.

This is passive replication because the operation is executed only on the primary node, and then state update messages are sent to the backups.

### These Are Both Examples of Primary Backup Replication

Irrespective of whether an active or passive strategy is used, both approaches shown here are examples of primary backup replication:

* The clients still communicate only with the primary
* The primary executes the required operation and communicates (in some way) with the backups
* The backups all acknowledge that they have successfully processed whatever message was sent to them
* Lastly, the primary commits the work itself and sends an acknowledgement back to the client

Exactly what type of message the primary sends to the backups is an internal implementation detail &mdash; as far as the external observer is concerned, it’s all primary backup replication

### Should We Choose Active or Passive?

So, what would cause us to choose one approach over the other?

Well, we need to consider factors such as the size of the resulting updated state, and the cost of the computation needed to derive that updated state.

For instance, what should we do if the primary receives the operation *"Increment everyone's account balance by one cent"* &mdash; and we have a million bank accounts?  In this case, the size of the state change would be huge, so passive backup would not be a good approach.

The following factors should be evaluated when deciding between active and passive replication:

* The update requires me to run a computation that costs `t` units of time
* I have `n` nodes on which this update must be applied
* The new state created by the computation is `s` bytes in size
* The network connecting the nodes transmits data at a rate of `r` bytes per unit of time

The question then boils down to crunching these numbers in order to work out which option gives me the quickest result.  In general, then:

* If an operation results in a large state change, then active replication is probably going to be better because sending the operation over the network uses up much less bandwidth than sending the changed stated
* If the cost of an operation is high, then passive replication is probably going to be better because the cost of computation is incurred only once (on the primary)

Active replication is also known as *"State Machine Replication"*

---

| Previous | Next
|---|---
| [Lecture 15](./Lecture%2015.md) | [Lecture 17](./Lecture%2017.md)

