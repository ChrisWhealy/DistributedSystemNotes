# Distributed Systems Lecture 9

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 17th, 2020 via [YouTube](https://www.youtube.com/watch?v=utsDozs1ZMc)

## Big Picture View of the Chandy-Lamport Algorithm

### Chandy-Lamport Assumptions

The Chandy-Lamport algorithm was the very first to define how you can take a reliable snapshot of a running distributed system.  This algorithm does make some fairly large assumptions, but that fact that it works even if marker messages are delayed is a really big achievement.

#### FIFO Delivery

We have assumed that the communication channels used between processes in a distributed system operate as FIFO queues.  This means that a channel is a mechanism for delivering an ordered sequence of messages.  But as we saw in the previous lecture, this is far more than an assumption - it is a requirement.  If channels did not behave as FIFO queues, then the Chandy-Lamport Snapshot Algorithm will break.

So, this requirement then leads to the question:

> ***Question***
> 
> If the delivery mechanism in a distributed system ***cannot*** guarantee ordered delivery (I.E. FIFO anomalies are possible), then are other algorithms available for taking a global snapshot?
> 
> ***Answer***
> 
> Yes, there are, but such algorithms have other drawbacks such as needing to pause application processes while the snapshot is taking place.

One particularly nice thing about the Chandy-Lamport algorithm is that you can take a snapshot while the application is running (I.E. it’s not a *stop-the-world* style algorithm).  The fact that a process sends out marker messages during its snapshot does not interfere with the application messages already travelling through the system.

#### Reliable Delivery

The Chandy-Lamport algorithm requires that messages are never:

* Lost
* Corrupted
* Duplicated

#### Processes Don't Crash

The Chandy-Lamport algorithm not robust against processes crashing whilst the snapshot is being taken.  If this were to happen, at best the snapshot would be incomplete; but to obtain a full snapshot, you would have to start the snapshot process over again.

### The Chandy-Lamport Algorithm is Guaranteed to Terminate

Without giving a formal proof, in order to take a snapshot of a distributed system we must record the both state of every process in the system, and the state of all the channels between those processes.

This overall task is accomplished by making each process responsible for recording:

* Its own internal state
* The state of all messages on its incoming channels

Then, when all the processes in the system have taken their own snapshot, the combined individual snapshots will form a coherent snapshot of the entire system.

If it can be demonstrated that these actions will terminate for an individual process, then it follows that it will terminate for the entire distributed system.

In section 3.3 of Chandy & Lamport's [original paper](https://lamport.azurewebsites.net/pubs/chandy.pdf) they say:

> If the graph is strongly connected and at least one process spontaneously records its state, then all processes will record their states in finite time (assuming reliable delivery)

The term ***strongly connected*** means that every process is reachable from every other process via channels.  In our example however, we have assumed that every process is directly connected to every other process by a channel, thus we not only have a strongly connected graph, we have a ***complete graph***.

If we draw the processes in our example system as a graph, every process is connected to every other process, making a total graph.

![Total Graph](./img/L9%20Total%20Graph.png)

However, Chandy & Lamport require only that the graph is strongly connected; for example, like this:

![Connected Graph](./img/L9%20Connected%20Graph.png)

## Simultaneous Snapshot Initiators

It is interesting that Chandy & Lamport state that ***at least*** one process must start the global snapshot by recording its own state.  They do not require that ***only*** one process initiates the global snapshot.

So, let's look at what happens when two processes simultaneously decide to take a snapshot.

IN the following diagram, processes `P1` and `P2` both decide to snapshot themselves.

![Simultaneous Snapshot 1](./img/L9%20Simultaneous%20Snapshot%201.png)

Since each is acting as the initiator, they both follow these rules:

* `P1` and `P2` both record their own state, creating states `S1` and `S2` respectively
* `P1` and `P2` both start recording messages on their incoming channels - <code>C<sub>21</sup></code> and <code>C<sub>12</sup></code> respectively
* `P1` and `P2` both send out marker messages on their outgoing channels

Now the marker messages arrive at each process.

![Simultaneous Snapshot 2](./img/L9%20Simultaneous%20Snapshot%202.png)

As soon as the marker messages arrive:

* `P1` and `P2` stop recording messages on their incoming channels
* `P1` and `P2` save the channel state of each recorded channel.  
    In this case, no messages arrived on <code>C<sub>21</sup></code>, but message `m` arrived on <code>C<sub>12</sup></code>

Again, we now have a coherent snapshot of the whole system in spite of the fact that two processes simultaneously decided to act as initiators.

This is known as a ***consistent cut*** - something we'll talk about soon.

### Is It Possible to Get a Bad Snapshot?

Could we end up with a bad snapshot such as the one shown below?

![Bad Snapshot](./img/L9%20Bad%20Snapshot.png)

No, this is in fact impossible because as soon as a process records its own state, it must immediately send out marker messages.  This is the rule that makes it impossible for event `B` to send message `m` to `P2`.  As soon as the internal state of `P1` is recorded, the marker message will be sent to `P2`, and because channels are FIFO queues, it is impossible for message `m` to arrive at `P2` before the marker message arrives.

It is vital therefore to understand the importance of this feature of the Chandy-Lamport algorithm.

To understand this, let's consider what would happen if simultaneous initiators were ***not*** permitted:

* `P1` decides it wants to take a snapshot
* `P1` sends a message to all the participants in the system saying *"Hi guys, I'd like to take a snapshot.  Is that OK with you?"*
* But whilst `P1` is waiting for everyone to respond, another process sends out a message saying *"Hi guys, I'd like to take a snapshot.  Is that OK with you?"*

This all gets very chaotic and might possibly lead to some sort of deadlock.

So, if multiple initiators are not permitted, then there has to be some way for processes to decide who is going to act as the sole initiator.  This then leads into the very challenging problem domain called ***Agreement Problems*** (Warning: Here be dragons!)

This makes the Chandy-Lamport algorithm very much easier to implement, because we do not have to care about solving the hard problem of agreeing on who will act as the initiator &mdash; any and all processes can snapshot whenever they like!

Further to this, any process that receives a marker message does not need to know either who sent that marker, or which process originally acted as the initiator.  Hence, marker messages can be very lightweight messages that do not need to carry any identifiers.

This is an example of a decentralised algorithm in that it can have multiple initiators.  There are, however, algorithms that are centralised, and these ***do*** require a single process to act as the initiator (We'll talk more about this type of algorithm later in the course).

## Why Do We Want Snapshots in the First Place?

What are snapshots good for?  Here are some ideas:

* ***Checkpointing***  
    In the event of failure, a checkpoint provides us with a reasonable restart point without having to start the whole calculation over again
* ***Deadlock detection***  
    Once a dead lock occurs at time `T` in a system, unless it is resolved, that deadlock will continue to exist for all points in time greater than `T`.  A snapshot can be used to perform deadlock detection
* ***Stable Property Detection***  
    A deadlock is an example of a ***Stable Property***.  A property of the system is said to be ***stable***, if, once it becomes true, it remains true.  
    Another example of a stable property is when the system has finished doing useful work (it might require human intervention to detect this) - I.E. Termination

Do not conflate ***deadlock*** with a process crashing.  Typically (but not always), a deadlock occurs when two running process enter a wait state for a response from the other.  Thus, neither process has crashed, but at the same time, neither process is capable of doing  any useful work because of the mutually dependent wait state.

## Chandy-Lamport Algorithm and Causality

The set of events in a Chandy-Lamport Snapshot will always make sense with respect to the causality of those events.

Now we need a new piece of terminology:  A ***cut*** is a time frontier that divides a Lamport diagram into past and future events.

![Cut](./img/L9%20Cut.png)

An event is said to be ***in the cut*** if it belongs to the past.  In Lamport diagrams where  time goes downwards, this means above the line.

So, for a cut to be ***consistent***, for all the events `E` in the cut, if `F->E` then `F` is also in the cut.

In the following diagrams `B->D`.  In other words, since event `B` happens before event `D`, any cut must preserve the fact that event `B` happens in the causal history of event `D`.

So, this cut is valid and is therefore called a consistent cut:

![Consistent Cut 1](./img/L9%20Consistent%20Cut%201.png)

Even though the cut has separated events `B` and `D`, their causality has not been reversed: event `B` remains on the past side of the cut, and event `D` remains on the future side.

However, this is an inconsistent cut:

![Inconsistent Cut](./img/L9%20Inconsistent%20Cut.png)

This cut is inconsistent because the causality of events `B` and `D` has been reversed.

As far as the data in this cut is concerned, the relation `B->D` is now broken.  This inconsistent cut has moved the future event `D` to the past, without recording the event that caused it.  Similarly, the past event `B` is seen as belonging to the future, without any connection to the future event it gives rise to.

## The Chandy-Lamport Algorithm Determines a Consistent Cut

If you take a cut of a distributed system and discover that the set of events in that cut is identical to a Chandy-Lamport snapshot, then you have a consistent cut.  This is what is meant when we say that a Chandy-Lamport Snapshot determines a consistent cut.

Going back to the more detailed example used in the previous lecture, we can see that the snapshot formed from three process states and six channel states forms a consistent cut.

![Consistent Cut 2](./img/L9%20Consistent%20Cut%202.png)

Another way of saying this is that a Chandy-Lamport snapshot is causally correct


## Recap: Delivery Guarantees and Protocols

### FIFO Guarantee

If a process sends message `m2` after `m1`, then any processing delivering both of these messages must deliver `m1` first.

A violation of this guarantee is a FIFO anomaly:

![FIFO Anomaly](./img/L5%20FIFO%20Anomaly.png)

### A Protocol to Enforce FIFO Delivery

In order to enforce FIFO delivery, we could implement strategies such as:

* Giving each message a sequence number.  The receiving process is then required to deliver these messages in sequence number order
* Requiring the receiving process to acknowledge receipt of a message

### Causal Delivery

The causal delivery guarantee says that if the send of message `m1` happens before the send of message `m2`, then `m1` must be delivered before `m2`.

A causal anomaly looks like this:

![Causal Anomaly](./img/L3%20Causal%20Anomaly.png)

### A Protocol to Enforce Causal Delivery

In order to enforce causal delivery (at least in the case of broadcast messages), we could implement a causal broadcast strategy based on vector clocks.  Each process is then responsible for maintaining its own vector clock and by means of a queueing mechanism, ensures that no ***messages from the future*** are delivered early.

### Total Delivery

If a process delivers `m1` then `m2`, all participating processes delivering both messages must deliver `m1` first.

A violation of totally ordered delivery looks like this:

![Total Order Anomaly](./img/L6%20Total%20Order%20Anomaly.png)

### A Protocol to Enforce Totally Ordered Delivery

We did not actually talk about a protocol for enforcing totally ordered delivery - we just spoke about it be hard.

But here's an idea.  If a fifth process were added to act as a coordinator, then totally ordered delivery could be ensured by telling every client process that in order to change the state of the data in the replicated databases, it must first check in with the coordinator.  The coordinator then acts as a middleman for ensuring that message delivery happens in the correct order.

This approach has several downsides:

* The coordinator process becomes a bottleneck making the system slow
* If the coordinator crashes, then updates stop until such time as the coordinator can restart

## Safety and Liveness Properties

All the properties we've spoken about here are known as a ***safety*** property or a  ***liveness** property.

| Safety Property | Liveness Property |
|---|---|
| Something bad will ***not*** happen | Something good ***eventually*** happens
| In a finite execution, we can demonstrate that something bad will happen if this property is not satisfied.<br><br>FIFO anomalies, Causal Anomalies and Totally Ordered Anomalies are all examples of safety properties | For example, all client messages are eventually answered.<br><br>However, because liveness properties are open-ended, we can provide a counter example as part of a ***finite*** execution, since the possibility of having to wait forever is not excluded.  This is why liveness properties are considered harder to reason about.


