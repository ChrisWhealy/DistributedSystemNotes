# Distributed Systems Lecture 7

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 13th, 2020 via [YouTube](https://www.youtube.com/watch?v=uJ62T48ZdBs)

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

1. If a message sent by process `P1` is delivered by process `P2`, increment the `P1` position in `P2`'s local clock
1. Before sending a message, a process must increment its own position in its local clock.  The new local clock value is then sent with the message as metadata.
1. Process `P2` should only deliver a message `m` received from `P1`, if the timestamp (I.E. vector clock) on message `m` conforms to the following two rules:
    
    * The vector clock value for process `P1` carried by message `m`, must be exactly one bigger than the vector clock value for process `P1` held by the receiving process `P2`.  Or written more algebraically:  

        <code>VC<sub>m</sub>[P1] = VC<sub>P2</sub>[P1] + 1</code>

        ***and***

    * The vector clock values for all other positions in message `m` must be less than or equal to the corresponding positions in `P2`'s vector clock.  Or, for all positions `k` in the vector clocks where `(k ≠ P1)`:


        <code>VC<sub>m</sub>[P<sub>k</sub>] ≤ VC<sub>P2</sub>[P<sub>k</sub>]</code>

### What Do These Rules Mean?

There are two important things to remember here:

1. We are only incrementing vector clock values on message-***send*** events, not message-receive events.  
    The value in each vector clock position is simply a count of the number of messages that process has sent so far.
1. These rules only apply to ***broadcast*** messages.  
    This means that every process in the system will (eventually) receive every message sent by every other process.  These rules do not apply for point-to-point messages!

***Rule 1:***&nbsp;&nbsp;<code>VC<sub>m</sub>[P1] = VC<sub>P2</sub>[P1] + 1</code>

Knowing this, we can understand the above rule to mean that in order to avoid creating a causal anomaly (I.E. by delivering a message out of order),  the receiver's local clock value for the sender must be exactly one smaller than the sender's clock value received in the message.  In other words, the timestamp on the message makes it the receiver's next expected message.

***Rule 2:***&nbsp;&nbsp;<code>VC<sub>m</sub>[P<sub>k</sub>] ≤ VC<sub>P2</sub>[P<sub>k</sub>]</code>

The second rule means that the number of message-sends performed by all the other processes in the system (I.E. the vector clock values) must be no bigger than the values recorded in the receiver's local clock.  In other words, the receiver has a complete record of all broadcast messages sent in the system; no messages are missing.

This is the rule that would be violated if `Carol` tried to deliver `Bob`'s rude "Up yours!" ***message from the future*** at the time it was received.

The vector clock on the message sent from `Bob` to `Carol` is `[1,1,0]`, but `Carol`'s vector clock is `[0,0,0]`.  The `1` in `Bob`'s position is correct because he is sending the message, and it is one greater than `Carol`'s local value, but the `1` in `Alice`'s position is a problem.  According to this, `Alice` has sent out a broadcast message, but `Carol` has no record of it &mdash; yet.

Since `1` is not ≤ `0`, `Carol` concludes that this newly arrived message has been received out of order and therefore must be queued until such time as we receive the delayed message from `Alice`.

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

If we assign vector clocks to these send-events and then apply the rules of causal broadcast, we will discover that causal delivery cannot rule out total order anomalies.

![Casual Delivery Cannot Prevent Total Order Anomaly](./img/L7%20TO%20Anomaly.png)

This is because, if we look back at the hierarchy of delivery guarantees, we can see that Total Order exists on its own branch from Casual and FIFO.

![Delivery Hierarchy](./img/L6%20Delivery%20Hierarchy%202.png)

So how can we rule out causal anomalies and, at the same time, ensure total order?

Well, we would need something more than vector clocks, or at least if we want causal order and also maintain total order, then we will need something more than this causal broadcast algorithm we have just described.

In general, it’s pretty annoying to have to enforce total order, so it’s much easier not to enforce it unless you really have to.

## Ways That Potential Causality is Used in Distributed Systems

The ***happens before*** relation is an example of potential causality.

We have already spoken of two of them:

- Ordering of events can be determined after the fact by assigning vector clocks (useful for debugging)
- Causal ordering of events as they happen (E.G. Using vector clocks to achieve causal broadcast)

These techniques provide ordering guarantees that prevent causal anomalies.

Another thing we have not mentioned yet is something called ***Consistent Global Snapshot***.  This is related to the first point above and is a way to obtain a picture of the global state of a distributed system.  However, this is far from trivial to implement because not only does every process in a system has its own state, every process also has its own idea of the state of every other process.

One thing we can say is that:

> If `A->B` and `B` is in the snapshot, then we should also expect to find `A` in the snapshot

So, the ***Consistent Global Snapshot*** is another important use of the *happens before* relation (or potential causality)

## How Do You Take a Global Snapshot of a Distributed System?

As has already been pointed out, in a distributed system, there really is no such things as an "observable" global state.  Each process has its own state that is the sum total of events that have happened in that process up until some particular point.  This includes the state of the process' internal memory.

So, we could lasso all the events and internal variables of a process and call that the state...

![Process State](./img/L7%20Process%20State.png)

But what about the state of all the other processes in the system?

One approach might be to use a global clock and inform every process that at a certain time of day (say `09:20`), every process must take a snapshot of itself.  However, even this approach won't work reliably because synchronising system clocks between computers is a notoriously difficult task.

![Snapshot Anomaly Caused by Using a Wall clock](./img/L7%20Wallclock%20Snapshot%20Anomaly.png)

We now have two inconsistent snapshots.

`P1` takes a snapshot of itself when its clock reaches `09:20`.  It then sends a message to `P2` informing it that a snapshot has been taken.  However, `P2`'s clock is running slightly slower than `P1`'s, so the `{p1:snapshot}` message receive event happens just before `P2`\'s clock ticks over to `09:20` and is therefore included in `P2`'s snapshot.

In spite of the fact that both snapshots were supposedly taken at `09:20`, they are in fact inconsistent with each other.  This is because `P1`'s message-send event `E5` is missing from `P1`'s snapshot but contained within `P2`'s snapshot.  Looking at the the other way around, `P2`'s snapshot contains a message-receive event, for which there is no corresponding send-event in `P1`'s snapshot.

As we can see, using the time-of-day clock is an error-prone approach to taking snapshots.

So, we need an algorithm that allows us to take a consistent global snapshot.

## The Chandy-Lamport Algorithm

We now introduce some new terminology:  ***Channels***.

A channel is simply a unidirectional connection between two processes.

### Channel Naming Convention

The channel from `P1` to `P2` is called <code>C<sub>12</sub></code>

The channel from `P2` back to `P1` is called <code>C<sub>21</sub></code>

![Channels 1](./img/L7%20Channels%201.png)

`P1` sends a message at event `A` over channel <code>C<sub>12</sub></code>.  This is received by `P2` at event `B`.

`P2` then sends a message back to `P1` at event `C` over channel <code>C<sub>21</sub></code>, but since this message has not yet arrived at `P1`, it is said to be ***in the channel***.

In the diagram above, channel <code>C<sub>12</sub></code> is said to be empty because there are no messages currently in flight, and channel <code>C<sub>21</sub></code> contains one message.

Here, we will assume that message channels act like FIFO queues (thus preventing FIFO violations).

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

So, the above snapshot forms a complete history of the events that occurred in these two processes; but what abou the following two snapshots - do they make sense?

![Bad Snapshots](./img/L7%20Bad%20Snapshot.png)

But what about this?  Is this a legal snapshot?

![Good Snapshot](./img/L7%20Good%20Snapshot.png)

Yes, this is perfectly valid - it was just taken prior to event `C` in `P2` happening.

### Is This a Snapshot of the Entire System?

A snapshot of the entire system is derived by a separate process that goes around collecting all the individual process snapshots and stitching them together to form of overall global snapshot.

The Chandy-Lamport algorithm actually predates the invention of vector clocks.  It has been designed to ensure that the entire event history of each process in the system is recorded without any violation of the ***happens before*** relation.




