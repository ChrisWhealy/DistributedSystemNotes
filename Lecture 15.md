# Distributed Systems Lecture 15

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 4th, 2020 via [YouTube](https://www.youtube.com/watch?v=UCmAzWvrFmo)


## Course Admin...

...snip...

Read Amazon's [Dynamo](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) paper and Google's [MapReduce](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) paper.

...snip...

### Exam Question: Chandy-Lamport Snapshot Bug

The following diagram shows a buggy implementation of the Chandy-Lamport snapshot algorithm.

Process `P2` initiates the snapshot, but then something goes wrong.  Where's the bug?

![Chandy-Lamport Snapshot Bug](./img/L15%20Chandy-Lamport%20Snapshot%20Bug.png)

The Chandy-Lamport algorithm assumes FIFO delivery of all messages &mdash; irrespective of whether they are application or marker messages; so, if we trace through the steps shown in the diagram, we can discover the bug:

* `P2` initiates the snapshot so it records its own state (the green ellipse around event `E`), then immediately sends out a marker message to `P1`
* `P1` receives the marker message and immediately records its own state (the green ellipse around events `A`, `B`, `C`, and `D`) and then sends out its marker message
* After `P2` sends out its marker message, its snapshot is complete, and it continues processing events in the normal way &mdash; resulting in event `F` sending out an application message to `P1`.

The bug is created by the fact that this diagram shows a FIFO anomaly created when the application message from `P2` event `F` ***overtakes*** the snapshot marker message.

As a result, `P1` event `D` is recorded in `P1`'s snapshot, but the event that caused it (`P2` event `F`) is missing from `P2`'s snapshot.  Thus, our snapshot is not a ***consistent cut***.

Remember that for a cut to be consistent, it must contain ***all*** events that led up to a certain point in time.  So, the inclusion of event `D` in `P1`'s snapshot is the problem because this is effectively a ***message from the future***.

This is an example of a situation in which a FIFO anomaly (out of order message delivery) leads to a causal anomaly (an inconsistent cut).

## Paxos: The Easy Parts

At the end of the last lecture, our discussion of the Paxos Algorithm got us up to here:

![Paxos Consensus Reached](./img/L14%20Paxos%206.png)

This was a very simple run of Paxos involving:

* One proposer,
* Three acceptors, and
* Two learners

In this example, the proposer `P` sent out `prepare` messages to a majority of the acceptors, which in this case, was two out of three; however, it would be been equally valid for `P` to have sent `prepare` messages to all the acceptors.  In fact, doing so would be quite smart because it mitigates against message loss, because on balance, even if one message is lost, you have still communicated with the majority of acceptors.

The same idea applies when the proposer listens for `promise` messages coming back from the acceptors.  It only needs to hear from a majority of the acceptors before it can be happy.  Exactly who those acceptors are is not important, and if it does hear back from all the acceptors then that's great, but it’s not a requirement.  It just needs to hear from a majority.

So, when we speak of a ***majority***, we are speaking of at least the ***minimum*** majority. For instance, if there are five acceptors, then the minimum majority is three: but if we hear back from four or all five, then this is not a problem.  The issue is that we must hear back from at least the minimum number of acceptors required to form a majority.

There are other subtleties involved in this algorithm that we will now go through, including what happens when there is more than one proposer.

## Milestones in the Paxos Algorithm

One thing that was mentioned in the previous lecture was that three specific milestones are reached during a run of the Paxos algorithm.  These are:

1. When the proposer receives `promise(n)` messages from a majority of acceptors.  

    ![Paxos Milestone 1](./img/L15%20Paxos%20Milestone%201.png)

    A majority of acceptors have all promised to respond to the agreed proposal number `n`; and by implication, they have also promised to ignore any request with a proposal number lower than `n`.

1. When a majority of acceptors all issue `accepted(n,val)` messages for proposal number `n` and some value `val`.  

    ![Paxos Milestone 2](./img/L15%20Paxos%20Milestone%202.png)

    Now, even though the other processes participating in the Paxos algorithm do not yet realise it, consensus has in fact been reached.
1. When the proposer(s) and learners receive `accepted(n,val)` messages from a majority of the acceptors.  

    ![Paxos Milestone 3](./img/L15%20Paxos%20Milestone%203.png)

    It is only now that the proposer(s) and the learners ***realise*** that consensus has already been reached


## Paxos: The Full Algorithm (Mostly)

A run of the Paxos algorithm involves the following sequence of message exchanges - primarily between the proposer and acceptors:

1. ***The Proposer***  
    Sends out `propose(n)` messages to at least the minimum number of acceptors needed to form a majority.  The proposal number `n` must be:
    *  Unique
    *  Higher than any previous proposal number used by ***this*** proposer

    It’s important to understand that the proposal number rules are applied to proposers ***individually***.  Consequently, if there are multiple proposers in the system, there does not need to be any agreement between proposers about what the next proposal number should be.
    
1. ***The Acceptor***  
    When the acceptor receives a `prepare(n)` message, it asks itself *"Have I already agreed to ignore proposals with this proposal number?"*.  If the answer is yes, then the message is simply ignored; but if not, it replies to the proposer with a `promise(n)` message.
    
    By returning a `promise(n)` message, the acceptor has now committed to ignore all messages with a proposal number smaller than `n`.  
    
1. ***The Proposer***  
    When the proposer has received `promise` messages from a majority of messages for a particular proposal number `n`, it sends an `accept(n,val)` message to a majority of acceptors containing both the agreed proposal number `n`, and the value `val` that it wishes to propose.
    
    Up till now, we have assumed that there is only one proposer &mdash; but next, we must examine what happens if there are multiple proposers.

1. ***The Acceptor***  
    When an acceptor receives an `accept(n,val)` message, it asks the same question as before: *"Have I already agreed to ignore messages with this proposal number?"*.  If yes, it ignores the message; but if no, it replies with an `accepted(n,val)` both back to the proposer ***and*** broadcasts this acceptance to all the learners.
    
### What Happens If There Is More Than One Proposer?

In this scenario, we will make two changes.  We will run the Paxos algorithm with two proposers, and for visual clarity, since learners do not actually take part in the steps needed to reach consensus, we will omit them from the from diagram.

Let's say we have ***two*** proposers <code>P<sub>1</sub></code> and <code>P<sub>2</sub></code> and as before, three acceptors (and we'll pretend we also have two learners)

Remember we previously stated that in situations where there are multiple proposers, these proposers must agree as to how they will ensure the uniqueness of their own proposal numbers.  So, in this case, we will assume that:

* Proposer <code>P<sub>1</sub></code> will use odd proposal numbers, and
* Proposer <code>P<sub>2</sub></code> will use even proposal numbers

So, proposer <code>P<sub>1</sub></code> sends out a `prepare(5)` message to a majority of the acceptors.  This is the first proposal number these acceptors have seen during this run of the protocol, so they are all happy to accept it and respond with `promise(5)` messages.

Proposer <code>P<sub>1</sub></code> is seeking consensus for value `1`, so it now sends out `accept(5,1)` messages and the majority of acceptors respond with `accepted(5,1)`

![Multiple Proposers 1](./img/L15%20Multiple%20Proposers%201.png)

Ok, that's fine; we seem to have agreed on value `1`.

But now, not knowing any of this has happened, proposer <code>P<sub>2</sub></code> decides to send out a `prepare(4)` message to all the acceptors...

![Multiple Proposers 2](./img/L15%20Multiple%20Proposers%202.png)

The `prepare(4)` message arrives at acceptors <code>A<sub>1</sub></code> and <code>A<sub>2</sub></code> ***after*** they have already agreed on proposal number `5`.  Since they are now ignoring proposal numbers less than `5`, they simply ignore this message.

Acceptor <code>A<sub>3</sub></code> however has not seen proposal number `4` before, so it happily agrees to it and sends back a `promise(4)` message to proposer <code>P<sub>2</sub></code>.

Proposer <code>P<sub>2</sub></code> is now left hanging.

It sent out `prepare` messages to all the acceptors but has only heard back from a minority of them.  The rest have simply not answered, and given the way asynchronous communication works, <code>P<sub>2</sub></code> has no idea ***why*** it has not heard back from the other acceptors.  They could have crashed, or they might be running slowly, or, as it turns out, the other acceptors have already agreed to have <code>P<sub>1</sub></code>'s babies...

So, all <code>P<sub>2</sub></code> can do is wait for its timeout period, and if it doesn't hear back within that time, it concludes that proposal number `4` was a bad idea and tries again.  This time, <code>P<sub>2</sub></code> shows up in a faster car (proposal number `6`)

![Multiple Proposers 3](./img/L15%20Multiple%20Proposers%203.png)

But wait a minute, consensus (milestone 2) has ***already*** been reached, so the acceptors now have a problem because:

* Acceptors cannot go back on their majority decision
* Acceptors cannot ignore `prepare` messages with a ***higher*** proposal number

So, here's where we must address one of the subtleties that we previously glossed over.

Previously, we stated only that if an acceptor receives a `prepare` message with a ***lower*** proposal number, it should simply ignore it.  Well, OK, that's fine.

But what about the case where we receive a proposal number that is ***higher*** than the last one?  Here is where we need to further qualify ***how*** that `prepare` message should be handled.

In this case, each acceptor must consider the following situation:

*"I've already promised to respond to proposal number `n`, but now I'm being asked to promise to respond to proposal number `n+1`"*

How the acceptor reacts now depends on what has happened in between it receiving the `prepare(n)` message and the `prepare(n+1)` message.

Either way, the acceptor cannot ignore the higher proposal number, so it is going to send out some sort of `promise` message; but this time, the acceptor must consider whether it has already accepted a value based on some earlier, lower proposal number.

* If no, then we accept the new proposal number with a `promise(n+1)` message as normal
* If yes, then we accept the new proposal number with a `promise(n+1, ...)` message, but in addition, we are obligated to tell the new proposer that we've already accepted a value using an older proposal number.

In the latter case, you can see that the `promise` message needs to carry some extra information.

In the above example, acceptor <code>A<sub>1</sub></code> has already agreed with proposer <code>P<sub>1</sub></code> that, using proposal number `5`, the value should be `1`; but now proposer <code>P<sub>2</sub></code> comes along and presents proposal number `6` to all the acceptors.

![Multiple Proposers 4](./img/L15%20Multiple%20Proposers%204.png)

So in this specific situation, acceptor <code>A<sub>3</sub></code> responds simply with `promise(6)` because although it previously agreed to proposal number `4`, nothing came of that, and it has not previously accepted any earlier value.

Acceptors <code>A<sub>1</sub></code> and <code>A<sub>2</sub></code> however, must respond with the message `promise(6,(5,1))`.

This extra information in the `promise` message effectively means: *"Ok, I'll move with you to proposal number `6` but understand this: using proposal number `5`, I already agreed to value `1`"*.

### So, What Should A Proposer Do with Such a Message?

Previously, we said that when a proposer receives `promise(n)` messages from a majority of acceptors, it will then send out `accept(n,val)` messages.  But here's where our description of the protocol needs to be refined.  What should the proposer do if it receives not a `promise(n)` message, but a <code>promise(n,(n<sub>old</sub>,val<sub>old</sub>))</code> message?

In our example, proposer <code>P<sub>2</sub></code> has received three `promise` messages:

* A straight-forward a `promise(6)` from <code>A<sub>3</sub></code>, and
* Two `promise(6,(5,1))` messages from <code>A<sub>1</sub></code> and <code>A<sub>2</sub></code>

Proposer <code>P<sub>2</sub></code> must now take into account that using proposal number `5`, the value `1` has already been agreed upon.

In this case, both `promise` messages contain the value `1` that was agreed upon using proposal number `5`; however, it is perfectly possible that <code>P<sub>2</sub></code> could receive multiple `promise` messages containing values agreed on by proposal numbers older than `5`.

So, the rule is this: proposer <code>P<sub>2</sub></code> must look at all the older, already agreed upon values, and chose the value corresponding to the most recent, old proposal number.

This is pretty ironic (and amusing) really because proposer <code>P<sub>2</sub></code> now has no choice over what value to propose.  It is constrained to propose the one value upon which consensus was most recently reached!  So, the fact that it wants to send out its own proposal is somewhat redundant, because the only value it can propose is one upon which consensus has already been reached...

So, now we must revise rule 3 given above.  Previously we stated:

>  When the proposer has received `promise` messages from a majority of messages for a particular proposal number `n`, it sends an `accept(n,val)` message to a majority of acceptors containing both the agreed proposal number `n`, and the value `val` that it wishes to propose.

But now we understand that the proposer does not have complete liberty to send out the value ***it wishes*** to propose; instead, it must first consider:

* If I have received any `promise` messages containing old agreed values, then I am obligated to propose the value belonging to the highest, old proposal number
* If I have received only simple `promise(n)` messages, then I am free to propose any value I like

So now, <code>P<sub>2</sub></code> can only send out the message `accept(6,1)`.

![Multiple Proposers 5](./img/L15%20Multiple%20Proposers%205.png)

Notice that <code>P<sub>2</sub></code> has not had to use the earlier proposal number `5`, but it was constrained to propose the value `1`, because this value has already been agreed upon.

So, what do the acceptors do now?  They simply invoke rule 4 above and respond with `accepted(6,1)`.

![Multiple Proposers 6](./img/L15%20Multiple%20Proposers%206.png)

Let's isolate the messages that were exchanged between proposer <code>P<sub>2</sub></code> and acceptor <code>A<sub>3</sub></code>.

![Multiple Proposers 7](./img/L15%20Multiple%20Proposers%207.png)

<code>A<sub>3</sub></code> only sees the following exchange of messages.

* <code>P<sub>2</sub></code> first tried proposal number `4`, but nothing came of that
* <code>P<sub>2</sub></code> tried again with proposal number `6`
* <code>A<sub>3</sub></code> went with the highest proposal number (`6`) and subsequently agreed to accept value `1`

As far as <code>A<sub>3</sub></code> is concerned, it thinks that value `1` was <code>P<sub>2</sub></code>'s idea.  It has no clue that <code>P<sub>2</sub></code> was proposing a value agreed upon by others.

