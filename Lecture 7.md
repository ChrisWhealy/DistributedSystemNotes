# Distributed Systems Lecture 7

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 13<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=uJ62T48ZdBs)

| Previous | Next
|---|---
| [Lecture 6](./Lecture%206.md) | [Lecture 8](./Lecture%208.md)


## Causal Broadcast Using Vector Clocks

Here's the classic causal anomaly again.

![Causal Anomaly](./img/L3%20Causal%20Anomaly.png)

The problem here is that the message `Carol` receives first is also delivered first.   The anomaly occurs because of out of order ***delivery***, not out of order ***receipt***.

The use of vector clocks helps eliminate the problem of causal anomalies because by comparing the vector clock value on the incoming message with its own vector clock, the receiver can decide whether or not the message should be delivered.

![Causal Broadcast 1](./img/L7%20Causal%20Broadcast%201.png)

In this case, when `Bob` receives `Alice`'s message, its fine for him to deliver it because `Bob`'s vector clock of `[0,0,0]` differs only by one from the vector clock of the incoming message, and this difference is in `Alice`'s position.  Therefore, since it came from `Alice`, it is correct to conclude that this message is the next event in both `Alice` and `Bob`'s causal history.

`Bob` then sends out his rude broadcast message in reply to `Alice`, but `Carol` receives this reply before the original message from `Alice`.

![Causal Broadcast 2](./img/L7%20Causal%20Broadcast%202.png)

By comparing the vector clock value on `Bob`'s message with her own vector clock, `Carol` can determine that this message should be queued, and not delivered yet. `Carol` does this by noticing that `Bob`'s vector clock position is one greater than her vector clock position for him.  Since this message came from `Bob`, this difference is expected.

Ok, so far so good.

However, the message vector clock also has a `1` in `Alice`'s position &mdash; and this is not expected.

Remember that in this scenario, our vector clocks only count message-send events; so when `Carol` sees a `1` in `Alice`'s position, this means that some message send event has taken place in `Alice` that `Carol` does not yet know about (probably because the messages have been received out of order).  Hence, `Carol` can correctly infer that this is a ***message from the future*** and should therefore be queued.


When `Alice`'s delayed message finally arrives at `Carol`, this message carries a vector clock value of `[1,0,0]` which is what `Carol` expects, so this message can be correctly delivered.

![Causal Broadcast 3](./img/L7%20Causal%20Broadcast%203.png)

Now `Carol` examines her message queue and discovers that the message with vector clock `[1,1,0]` can now be delivered, because all the preceding messages in her causal history have been delivered in the correct order.

![Causal Broadcast 4](./img/L7%20Causal%20Broadcast%204.png)

## Rules for Causal Broadcast

1. Before sending a message to process `P2`, process `P1` records this send event by incrementing its own position in its local vector clock.  The updated vector clock is then sent with the message as metadata.
1. At such time as `P2` delivers `P1`'s message, `P2` increments the `P1` clock value in its own local vector clock (I.E. it records the delivery of `P1`'s message)
1. Process `P2` should only deliver `P1`'s message if the clock value received with that message conforms to the following two rules:
    
    * The clock value for process `P1` in the message must be exactly one bigger than the clock value for process `P1` in the receiving process `P2`.  Or written more algebraically:  

        <code>VC<sub>msg</sub>[P1] = VC<sub>P2</sub>[P1] + 1</code>

        ***and***

    * The message clock values for all other positions must be less than or equal to the corresponding positions in `P2`'s vector clock.  Or, for all positions `k` in the vector clock where `(k ≠ P1)`:

        <code>VC<sub>msg</sub>[P<sub>k</sub>] ≤ VC<sub>P2</sub>[P<sub>k</sub>]</code>

### What Do These Rules Mean?

There are two important things to remember here:

1. We are only incrementing vector clock values on message-***send*** events, not message-receive events.  
    The value in each vector clock position is simply a count of the number of messages that process has sent so far.
1. These rules only apply to ***broadcast*** messages, not point-to-point messages!  
    This means that every process in the system will (eventually) receive every message sent by every other process.

***Rule 1:***&nbsp;&nbsp;<code>VC<sub>msg</sub>[P1] = VC<sub>P2</sub>[P1] + 1</code>

Knowing this, we can understand the above rule to mean that in order to avoid creating a causal anomaly (I.E. by delivering a message out of order),  the receiver's local clock value for the sender must be exactly one smaller than the sender's clock value received in the message.

***Rule 2:***&nbsp;&nbsp;<code>VC<sub>msg</sub>[P<sub>k</sub>] ≤ VC<sub>P2</sub>[P<sub>k</sub>], (k ≠ P1)</code>

The second rule means that the number of message-sends performed by all the other processes in the system (I.E. the vector clock values) must be no bigger than the values recorded in the receiver's local clock.  In other words, the receiver keeps a complete record of all broadcast messages sent in the system; no message-send events are missing.

This is the rule that would be violated if `Carol` tried to deliver `Bob`'s rude "Up yours!" ***message from the future*** at the time it was received.

The vector clock on the message sent from `Bob` to `Carol` is `[1,1,0]`, but `Carol`'s vector clock is `[0,0,0]`.  The `1` in `Bob`'s position is correct because he is sending the message, and it is one greater than `Carol`'s local value, but the `1` in `Alice`'s position is a problem.  According to this, `Alice` has sent out a broadcast message, but `Carol` has no record of it &mdash; yet.

Since `1` is not ≤ `0`, `Carol` correctly concludes that this newly arrived message has been received out of order and therefore must be queued until such time as we receive the delayed message from `Alice`.

### Another Example

Here. `Alice` sends a broadcast message to `Bob` and `Carol` saying ***"I lost my wallet :-("***

![Causal Broadcast 5](./img/L7%20Causal%20Broadcast%205.png)

In the absence of any other messages, its fine for both `Bob` and `Carol` to deliver this message.

A little later, `Alice` finds her wallet and tells `Bob` and `Carol` about this happy event.  However, `Alice`'s message to `Carol` gets delayed.

![Causal Broadcast 6](./img/L7%20Causal%20Broadcast%206.png)

It’s fine for `Bob` to deliver this message because it came from `Alice` and is the next expected message from her.  `Bob` is very happy that `Alice` has found her wallet and sends out a broadcast message to both `Alice` and `Carol` saying how pleased he is.

Unfortunately, `Alice`'s original message doesn't reach `Carol` until after `Bob`'s response arrives.

![Causal Broadcast 7](./img/L7%20Causal%20Broadcast%207.png)

Fortunately, `Carol` recognises that `Bob`'s response is a ***message from the future***, and places it in her message queue.  Otherwise, she would think that `Bob` is a complete jerk because as far as `Carol` is concerned, `Alice` has just lost her wallet and `Bob` seems to be really happy about this!

![Causal Broadcast 8](./img/L7%20Causal%20Broadcast%208.png)

Only after `Carol` delivers the preceding message, does she then deliver the message that arrived too early.

## Can Vector Clocks Establish a Total Order on Messages?

Remember what a total order anomaly looks like:

![Total Order Anomaly](./img/L6%20Total%20Order%20Anomaly.png)

If we assign vector clocks to these send-events and then apply the rules of causal broadcast, we will discover that causal delivery cannot rule out total-order anomalies.

![Casual Delivery Cannot Prevent Total Order Anomaly](./img/L7%20TO%20Anomaly.png)

This is because, if we look back at the hierarchy of delivery guarantees, we can see that Total Order exists on its own branch from Casual and FIFO.

![Delivery Hierarchy](./img/L6%20Delivery%20Hierarchy%202.png)

So how can we rule out causal anomalies and, at the same time, ensure total order?

Well, if we want to maintain both causal order ***and*** total order, then we will need something more than vector clocks and the causal broadcast algorithm we have just described.

In general, it’s pretty annoying to have to enforce total order, so it’s much easier not to enforce it unless you really have to.

## Ways That Potential Causality is Used in Distributed Systems

The ***happens before*** relation is an example of potential causality.

We have already spoken of two of them:

- Ordering of events can be determined after the fact by assigning vector clocks (useful for debugging)
- Causal ordering of events as they happen (E.G. Using vector clocks to achieve causal broadcast)

These techniques provide ordering guarantees that prevent causal anomalies.

## Consistent Global Snapshot

Another thing we have not mentioned yet is something called ***Consistent Global Snapshot***.  This is related to the first point above and is a way to obtain a picture of the global state of a distributed system.  However, this is far from trivial to implement because not only does every process in a system have its own state, every process also has its own idea of the state of every other process.

One thing we can say is that:

> If `A->B` and `B` is in the snapshot, then we should also expect to find `A` in the snapshot

So, the ***Consistent Global Snapshot*** is another important use of the *happens before* relation (or potential causality)

## How Do You Take a Global Snapshot of a Distributed System?

As has already been pointed out, in a distributed system, there really is no such things as an "observable" global state.  Each process has its own state that represents the sum total of activity in that process up until some particular point.  This includes the state of the process' internal memory.

So, we could lasso all the events and internal variables of a process and call that the state...

![Process State](./img/L7%20Process%20State.png)

But what about the state of all the other processes in the system?

One approach might be to use a global clock and inform every process that at a certain time of day (say `09:20`), they must all take a snapshot of themselves.  However, as we have already seen in [lecture 2](https://github.com/ChrisWhealy/DistributedSystemNotes/blob/master/Lecture%202.md#time-and-how-we-measure-it), this approach won't work reliably &mdash; not because a process can't take a selfie (so to speak), but because synchronising clocks between computers is a notoriously difficult task.

![Snapshot Anomaly Caused by Using a Wall clock](./img/L7%20Wallclock%20Snapshot%20Anomaly.png)

Uh oh! Now we have a problem.  Remember what we said above:

> If `A->B` and `B` is in the snapshot, then we should also expect to find `A` in the snapshot

From the diagram, we can see that the message-send event `E` in `P1` causes event `W` in `P2`.

Therefore `E(P1) -> W(P2)`

But since `P2`'s clock is running slightly slower than `P1`'s, the `{p1:snapshot}` message arrives at `P2` (causing event `W`) just ***before*** `P2`'s clock ticks over to `09:20`; therefore, as far as `P2` is concerned, event `W` should be included in the snapshot.

Now when we compare the data in `P1`'s snapshot with the data in `P2`'s snapshot, we are unable to explain the cause of event `W` &mdash; it just pops up out of nowhere because there's a gap in its causal history.  And all of this happens in spite of both processes thinking they took their snapshots at `09:20`.

As we can see, using a machine's time-of-day clock is an error-prone approach to taking consistent snapshots.

So, we need an algorithm that allows us to snapshot an entire system ***consistently***.

## The Chandy-Lamport Algorithm

We now introduce some new terminology:  ***Channels***.

A channel is simply a unidirectional communication path between two processes.

### Assumptions

As an aside, the success of the Chandy-Lamport algorithm relies entirely on the truth of the following assumptions:

1. A message is ***guaranteed*** to arrive at the recipient &mdash; eventually
1. All channels act as FIFO queues, thus making it impossible for two messages to arrive out of order (I.E. we can exclude the possibility of FIFO anomalies)
1. Processes don't crash! (The topic of process failure is dealt with later in [lecture 10](./Lecture%2010.md))

### Channel Naming Convention

Between any two processes `P1` and `P2`, two channels exist.  Communication from `P1` to `P2` passes along channel <code>C<sub>12</sub></code>, and communication in the other direction passes along channel <code>C<sub>21</sub></code>

![Channels 1](./img/L7%20Channels%201.png)

`P1` sends a message to `P2` at event `A` over channel <code>C<sub>12</sub></code>.  This is received by `P2` at event `B`.

`P2` then sends a message back to `P1` at event `C` over channel <code>C<sub>21</sub></code>, but since this message has not yet arrived at `P1`, it is said to be ***in the channel***.

In the diagram above, channel <code>C<sub>12</sub></code> is said to be empty because there are no messages currently in flight, and channel <code>C<sub>21</sub></code> contains one message.

### Applying the Chandy-Lamport Algorithm for Taking a Snapshot

In this case, process `P1` sends the first message, and therefore we will call it the ***initiator process***.

![Channels 2](./img/L7%20Channels%202.png)

In the above diagram

1. Process `P1` sends a message out at event `A` on its only channel <code>C<sub>12</sub></code>
1. `P1` then takes a snapshot of itself (`S1`) and immediately sends out a special message called a ***marker*** on all its channels.
     Sending a marker message is actually part of the snapshot algorithm itself and must be the first thing done after a process records its own state. The marker messages themselves do not form part of the snapshot.
1. Process `P1` then starts to record all the messages it receives on all of its incoming channels

So, what happens when a process receives a marker message?

There are two cases:

1. If this is the first marker this process has seen:
    * It takes a snapshot of itself
    * It marks the channel on which it received the marker message as empty
    * It sends a marker message out on all of its outgoing channels

1. If this process has already seen a marker message before:
    * It stops recording all incoming messages on that channel
    * It sets that channel's state to be the sum of the events that arrived whilst recording was active 

So when `P2` receives the marker message from `P1`, it immediately records its state (`S2`), marks channel <code>C<sub>12</sub></code> as empty, and sends a marker out on all its channels (in this case, `P2` only has one channel)

![Channels 3](./img/L7%20Channels%203.png)

What does `P1` do when it sees `P2`'s marker message?

Well, has `P1` seen a marker message before?

Yes, it has.  By sending out the first marker message, `P1` is said to have seen a marker message.  So now `P1` stops recording on the incoming channel on which the marker message arrived.

Did `P1` record anything on this channel?

Let's say that it did.  Let's say that the message sent from `P2` at event `C` actually arrived at `P1` as event `D`.

![Channels 4](./img/L7%20Channels%204.png)

So, event `D` is now a recorded event and the snapshot is complete:

* We have recorded the state of `P1` in `S1`
* We have recorded the state of `P2` in `S2`
* We have recorded the incoming message event `D` in `P1`

So, the above snapshot forms a complete history of the events that occurred in these two processes; but what about the following two snapshots - do they make sense?

![Bad Snapshots](./img/L7%20Bad%20Snapshot.png)

But what about this?  Is this a legal snapshot?

![Good Snapshot](./img/L7%20Good%20Snapshot.png)

Yes, this is perfectly valid - it was just taken prior to event `C` in `P2` happening.

### Is This a Snapshot of the Entire System?

A snapshot of the entire system is derived by a separate process that goes around collecting all the individual process snapshots and stitching them together to form of overall global snapshot.

The Chandy-Lamport algorithm actually predates the invention of vector clocks.  It has been designed to ensure that the entire event history of each process in the system is recorded without any violation of the ***happens before*** relation.

---

| Previous | Next
|---|---
| [Lecture 6](./Lecture%206.md) | [Lecture 8](./Lecture%208.md)


