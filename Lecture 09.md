# Distributed Systems Lecture 9

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 17<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=utsDozs1ZMc)

| Previous | Next
|---|---
| [Lecture 8](./Lecture%2008.md) | [Lecture 10](./Lecture%2010.md)


## Big Picture View of the Chandy-Lamport Algorithm

### Chandy-Lamport Assumptions

The Chandy-Lamport algorithm was the very first to define how you can take a reliable snapshot of a running distributed system.
This algorithm does make some fairly large assumptions, but the fact that it works even if marker messages are delayed is a significant achievement.

#### FIFO Delivery

We have assumed that the communication channels used between processes in a distributed system operate as FIFO queues.
This means that a channel is a mechanism for delivering an ordered sequence of messages.
But as we saw in the previous lecture, this is far more than an assumption - it is a requirement.
If channels did not behave as FIFO queues, then the Chandy-Lamport Snapshot Algorithm would break.

So, this requirement then leads to the question:

***Q:***&nbsp;&nbsp; If the delivery mechanism in a distributed system ***cannot*** guarantee ordered delivery (I.E. FIFO anomalies are possible), then are other algorithms available for taking a global snapshot?  
***A:***&nbsp;&nbsp; Yes, there are, but such algorithms have other drawbacks such as needing to pause application processing while the snapshot is taking place.

One particularly nice thing about the Chandy-Lamport algorithm is that you can take a snapshot while the application is running (I.E. it’s not a *stop-the-world* style algorithm).
The fact that a process sends out marker messages during its snapshot does not interfere with the application messages already travelling through the system.

#### Reliable Delivery

The Chandy-Lamport algorithm requires that messages are never:

* Lost
* Corrupted
* Duplicated

#### Processes Don't Crash

The Chandy-Lamport algorithm is not robust against processes crashing whilst the snapshot is being taken.
If this were to happen, at best the snapshot would be incomplete; but to obtain a full snapshot, you would have to start the snapshot process over again.

### The Chandy-Lamport Algorithm is Guaranteed to Terminate

Without giving a formal proof, in order to take a snapshot of a distributed system we must make each process responsible for recording:

* Its own internal state
* The state of all messages on its incoming channels

Then, when all the processes in the system have completed their snapshots, the sum of the individual snapshots becomes the consistent snapshot of the entire system.

If it can be demonstrated that this approach for taking a snapshot will terminate for an individual process, and all processes follow the same approach, then it follows that the entire snapshot process will terminate for the entire distributed system.

In section 3.3 of Chandy & Lamport's [original paper](./papers/chandy.pdf) they say:

> If the graph is strongly connected and at least one process spontaneously records its state, then all processes will record their states in finite time (assuming reliable delivery)

The term ***strongly connected*** means that every process is reachable from every other process via channels.
In our example however, we have assumed that every process is directly connected to every other process by a channel, thus we not only have a strongly connected graph, we have a ***complete graph***.

If we draw the processes in our example system as a graph, every process is connected to every other process, making a total graph.

![Total Graph](./img/L9%20Total%20Graph.png)

However, Chandy & Lamport require only that the graph is strongly connected; for example, like this:

![Connected Graph](./img/L9%20Connected%20Graph.png)

`P3` can still send messages to `P2` but it must send them via `P1`.

## Simultaneous Snapshot Initiators

It is interesting that Chandy & Lamport state that ***at least*** one process must start the global snapshot by recording its own state.
They do not require that ***only*** one process initiates the global snapshot.

So, let's look at what happens when two processes simultaneously decide to take a snapshot.

In the following diagram, processes `P1` and `P2` simultaneously decide to snapshot themselves.

![Simultaneous Snapshot 1](./img/L9%20Simultaneous%20Snapshot%201.png)

Since each is acting as the initiator, they both follow these rules:

* `P1` and `P2` both record their own state, creating states `S1` and `S2` respectively
* `P1` and `P2` both start recording messages on their incoming channels - <code>C<sub>21</sup></code> and <code>C<sub>12</sup></code> respectively
* `P1` and `P2` both send out marker messages on their outgoing channels

Now the marker messages arrive at each process.

![Simultaneous Snapshot 2](./img/L9%20Simultaneous%20Snapshot%202.png)

As soon as the marker messages arrive:

* `P1` and `P2` stop recording messages on their incoming channels
* `P1` and `P2` save the state of each recorded channel.  
    In this case, no messages arrived on <code>C<sub>21</sup></code>, but message `m` arrived on <code>C<sub>12</sup></code>

Again, we now have a coherent snapshot of the whole system in spite of the fact that two processes simultaneously decided to act as initiators.

This is known as a ***consistent cut*** - something we'll talk about a little later.

### Is It Possible to Get a Bad Snapshot?

Could we end up with a bad snapshot such as the one shown below?

![Bad Snapshot](./img/L9%20Bad%20Snapshot.png)

No, this is in fact impossible because as soon as a process records its own state, it must immediately send out marker messages.
This is the rule that makes it impossible for message `m` to arrive at `P2` before the snapshot processing has completed.

The rules of the Chandy-Lamport Snapshot algorithm state that as soon as the internal state of `P1` is recorded, marker message ***must*** be sent on all outgoing channels.
So, the marker message will always be sent ***before*** event `B` happens in `P1` that sends message `m` to `P2`.
Also, because channels are FIFO queues, it is impossible for message `m` to arrive at `P2` before the marker message arrives.

To understand how important this is, let's consider what would happen if simultaneous initiators were ***not*** permitted and the process wishing to initiate a snapshot had to seek agreement from the other participants before starting:

* `P1` decides it wants to take a snapshot
* `P1` sends a message to all the participants saying *"Hi guys, I'd like to take a snapshot.  Is that OK with you?"*
* But whilst `P1` is waiting for everyone to respond, another process sends out a message saying *"Hi guys, I'd like to take a snapshot.  Is that OK with you?"*

This all gets very chaotic and could well lead to some sort of deadlock.

So, if multiple initiators are not permitted, then there has to be some way for processes to decide who is going to act as the sole initiator.
This then leads into the very challenging problem domain known as ***Agreement Problems*** (Warning: here be dragons!)

Since the Chandy-Lamport algorithm permits multiple initiators, it is very much easier to implement because we do not have to care about solving the hard problem of agreeing on either who will act as the initiator, or when a snapshot should be taken &mdash; ***any*** process can take a snapshot ***any*** time it likes!

Further to this, any process that receives a marker message does not need to care about either who sent that marker, or which process originally acted as the initiator.
Hence, markers can be very lightweight messages that do not need to carry any data such as the identity of the initiator.

The Chandy-Lamport algorithm is an example of a decentralised algorithm.
There are, however, algorithms that are centralised, and these ***do*** require a single process to act as the initiator (we'll talk more about this type of algorithm later in the course).

## Why Do We Want Snapshots in the First Place?

What are snapshots good for?  Here are some ideas:

* ***Checkpointing***  
    If we are performing a process that is either expensive or based on non-deterministic input (such as user requests arriving over a network), then in the event of failure, a checkpoint allows us to reconstruct the system's state and provides us with a reasonable restart point.  
    For expensive calculations, all our calculation effort up until the last checkpoint is preserved, and for transactional systems, the system state preserved at the checkpoint reduces data loss down to only those transactions that occurred between the checkpoint being taken and the system failing.
* ***Deadlock detection***  
    Once a dead lock occurs at time `T` in a system, unless it is resolved, that deadlock will continue to exist for all points in time greater than `T`.
    A snapshot can be used to perform deadlock detection and thus serves as a useful debugging tool
* ***Stable Property Detection***  
    A deadlock is an example of a ***Stable Property***.
    A property of the system is said to be ***stable***, if, once it becomes true, it remains true.  
    Another example of a stable property is when the system has finished doing useful work - I.E. Task termination (however, human intervention might be required in order to detect that this state has been reached) 

Be careful not to conflate a ***deadlock*** with a process crashing.
Typically (but not always), a deadlock occurs when two running processes enter a mutual wait state.
Thus, neither process has crashed, but at the same time, neither process is capable of doing any useful work because each is waiting for a response from the other.

## Chandy-Lamport Algorithm and Causality

The set of events in a Chandy-Lamport Snapshot will always make sense with respect to the causality of those events.

Now we need to introduce a new piece of terminology: a ***cut***.

A cut is a time frontier that divides a Lamport diagram into past and future events.

![Cut](./img/L9%20Cut.png)

An event is said to be ***in the cut*** if it belongs to the past.
In Lamport diagrams, where time goes downwards, this means events occurring above the line.

So, if event `E` is in the cut and `D->E` then for the cut to be ***consistent***, event `D` must also be in the cut.

This is a restatement of the principle of [consistent global snapshots](https://github.com/ChrisWhealy/DistributedSystemNotes/blob/master/Lecture%207.md#consistent-global-snapshot) that we saw in [lecture 7](./Lecture%207.md).

In both the of following diagrams `B->D`.
Therefore, for a cut to be consistent, it must preserve the fact that event `B` happens in the causal history of event `D`.

So, this cut is valid and is therefore called a consistent cut:

![Consistent Cut 1](./img/L9%20Consistent%20Cut%201.png)

Even though the cut has separated events `B` and `D`, their causality has not been reversed: event `B` remains on the "past" side of the cut, and event `D` remains on the "future" side.

However, in the diagram below, the cut is inconsistent because the causality of events `B` and `D` has been reversed:

![Inconsistent Cut](./img/L9%20Inconsistent%20Cut.png)

The happens-before relation between events `B` and `D` is now broken because the cut has moved the future event `D` into the past, and the past event `B` into the future.

## The Chandy-Lamport Algorithm Determines a Consistent Cut

If you take a cut of a distributed system and discover that the set of events in that cut is identical to a Chandy-Lamport snapshot, then you have a consistent cut.
This is what is meant when we say that a Chandy-Lamport Snapshot determines a consistent cut.

Going back to the more detailed example used in the previous lecture, we can see that the snapshot formed from three process states and six channel states forms a consistent cut.

![Consistent Cut 2](./img/L9%20Consistent%20Cut%202.png)

Another way of saying this is that a Chandy-Lamport snapshot is causally correct.

## Recap: Delivery Guarantees and Protocols

### FIFO Guarantee

If a process sends message `m2` after `m1`, then any process delivering both of these messages must deliver `m1` first.

A violation of this guarantee is a FIFO anomaly:

![FIFO Anomaly](./img/L5%20FIFO%20Anomaly.png)

### A Protocol to Enforce FIFO Delivery

In order to enforce FIFO delivery, we could implement strategies such as:

* Giving each message a sequence number.
The receiving process is then required to deliver these messages in sequence number order
* Requiring the receiving process to acknowledge receipt of a message

### Causal Delivery

The [causal delivery](https://github.com/ChrisWhealy/DistributedSystemNotes/blob/master/Lecture%206.md#causal-delivery) guarantee simply says messages must be delivered in the order they were sent.

> If `m1`'s send happens before `m2`'s send, then `m1`'s delivery must happen before `m2`'s delivery.

A causal anomaly looks like this:

![Causal Anomaly](./img/L3%20Causal%20Anomaly.png)

### A Protocol to Enforce Causal Delivery

In order to enforce causal delivery (at least in the case of broadcast messages), we could implement a causal broadcast strategy based on vector clocks.
Each process is then responsible for maintaining its own vector clock and by means of a queueing mechanism, ensures that no ***messages from the future*** are delivered early.

### Totally Ordered Delivery

If a process delivers `m1` then `m2`, all participating processes delivering both messages must deliver `m1` first.

A violation of totally-ordered delivery looks like this:

![Total Order Anomaly](./img/L6%20Total%20Order%20Anomaly.png)

### A Protocol to Enforce Totally-Ordered Delivery

We did not actually talk about a protocol for enforcing totally-ordered delivery - we just spoke about it being hard!

But here's an idea.
If a fifth process were added to act as a coordinator, then totally-ordered delivery could be ensured by telling every client process that in order to change the state of the data in the replicated databases, it must first check in with the coordinator.
The coordinator then acts as a middleman for ensuring that message delivery happens in the correct order.

However, this approach has several downsides:

* Under high-load situations, the coordinator process could become a performance bottleneck
* If the coordinator crashes, then all updates stop until such time as the coordinator can be restarted (but who's going to monitor the coordinator process &mdash; a coordinator-coordinator process?)

## Safety and Liveness Properties

Let's now briefly introduce the next topic, that of ***safety*** and ***liveness*** properties.

| Safety Property | Liveness Property |
|---|---|
| Something bad will ***not*** happen | Something good ***eventually*** happens
| In a finite execution, we can demonstrate that something bad will happen if this property is not satisfied.<br><br>FIFO anomalies, Causal Anomalies and Totally-Ordered Anomalies are all examples of safety properties because we can demonstrate that their failure causes something bad to happen | For example, all client messages are eventually answered.<br><br>The problem though is that liveness properties tend to have very open-ended definitions.  This means we might have to wait forever before something good happens... Consequently, when considering ***finite*** execution, it is very difficult (impossible?) to provide counter examples.<br><br>This is why liveness properties are much harder to reason about.

---

| Previous | Next
|---|---
| [Lecture 8](./Lecture%2008.md) | [Lecture 10](./Lecture%2010.md)

