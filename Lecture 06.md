# Distributed Systems Lecture 6

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 10<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=UoIiwJ2G2fc)

| Previous | Next
|---|---
| [Lecture 5](./Lecture%2005.md) | [Lecture 7](./Lecture%2007.md)

## Recap

### Sending, Receiving and Delivering

* ***Sending*** a message is active: you choose when and if to send a message
* ***Receiving*** a message is passive: you cannot choose either when or if a message will arrive.
    All you can do is react by capturing it
* ***Delivering*** a message is active: it is the conscious choice to act upon the contents of a received message.

But why would you want to queue a message before processing it?
Typically, because messages need to be processed in the correct order, which could well be different from the order in which they were received.

### FIFO Delivery

If a process sends message `M2` after `M1`, then any process delivering ***both*** of these messages must deliver `M1` first then `M2`.
Failure to do this constitutes a protocol violation as described in the previous lecture as a "FIFO anomaly"

![FIFO violation or anomaly](./img/L5%20FIFO%20Anomaly.png)

In this case, irrespective of the order in which process `Bob` received messages `m1` and `m2`, it should always ***deliver*** message `m1` first, followed by message `m2`.

In the case that `Bob` does not receive one (or either) of these messages, then no FIFO violation could have occurred, because here we are concerned with message ***delivery***, not message ***receipt***.


### FIFO Delivery Implementation

In real-life, it is unusual to have to implement FIFO delivery yourself because most distributed systems communicate using TCP which already implements FIFO packet delivery.

## Causal Delivery

There are different ways of phrasing this, but one way is to say:

> If `m1`'s send happens before `m2`'s send, then `m1`'s delivery must happen before `m2`'s delivery

Here's an example of such a violation

![Causal violation](./img/L6%20Causal%20Violation.png)

In process <code>P<sub>1</sub></code>,  event `A` happens in the causal history of event `B`; therefore, any messages sent from <code>P<sub>1</sub></code> to <code>P<sub>2</sub></code> should be processed in the same causal order as the events that generated them.

But now, let's go back to the "Bob smells" example used in [lecture 3](./Lecture%203.md)

![Causal Anomaly](./img/L3%20Causal%20Anomaly.png)

***Q:***&nbsp;&nbsp; Is this a FIFO violation?  
***A:***&nbsp;&nbsp; No (but only in a vacuous sense...)

The reason is that a FIFO violation only occurs when two messages ***from the same originating process*** are delivered out of order by the receiving process.

In the above diagram, there is no single process that sends ***two*** distinct messages to the same receiving process.  Here:

* `Alice` sends a single message to `Bob`
* `Alice` sends a single message to `Carol`
* `Bob` sends a single message to `Alice`
* `Bob` sends a single message to `Carol`
* `Carol` is confused...

However, this scenario is still a causal anomaly.  In general, at least three communicating processes are required to create a causal anomaly.

### Can a Message be Sent to Multiple Destinations?

Yes.
Messages can be sent either to a group of participants in a network (a multicast message), or to all participants in a network (a broadcast message).
The idea of broadcast messages is something that will be dealt with later.

## Totally-Ordered Delivery

This is another correctness property.

> If a process delivers message `M1` followed by `M2`, then ***all*** processes delivering both `M1` and `M2` must deliver `M1` first followed by `M2`.

Let's say we have two client processes `C1` and `C2` that each broadcast a message to two processes `R1` and `R2`.
In this scenario, processes `R1` and `R2` each maintain their own replica of some key/value store.

If processes `R1` and `R2` do not deliver the messages in the correct order, then we will encounter a violation that results in the replicas disagreeing with each other as to what the value of `x` should be.
In other words, this violation creates an inconsistency between data replicas.

![Total-Order Anomaly](./img/L6%20Total%20Order%20Anomaly.png)

This is known as a ***Total-Order Anomaly*** and is created when process `R1` delivers message `m1` followed by `m2`, but process `R2` delivers message `m2` followed by `m1`.

### Delivery Guarantees

Since we know that causal delivery also ensures FIFO delivery, we can start to arrange these delivery strategies in a hierarchy, with the weakest at the bottom.
Here, we will use the term `YOLO` to indicate the delivery guarantee that makes no guarantees!

![Delivery Hierarchy 1](./img/L6%20Delivery%20Hierarchy%201.png)

Where would Totally Ordered Delivery fit in to this scheme?

In fact, it would get its own branch because a FIFO anomaly is not necessarily an anomaly as far as Totally-Ordered Delivery is concerned.

We recall that a FIFO anomaly is the following:

![FIFO violation or anomaly](./img/L5%20FIFO%20Anomaly.png)

But since the definition of Totally-Ordered Delivery says that ***all*** processes delivering both `m1` and `m2` must do so in a consistent order, the above FIFO anomaly is not an anomaly for Totally-Ordered Delivery because there is only one receiving process.
So the order in which that process delivers the messages is immaterial.
Thus, this scenario only vacuously conforms to a Totally-Ordered Delivery.

Conversely, Totally-Ordered Delivery violations are not necessarily FIFO violations.

![Delivery Hierarchy 2](./img/L6%20Delivery%20Hierarchy%202.png)

### What Does This Hierarchy Imply?

This hierarchy helps us understand what we can and cannot expect out of a particular delivery guarantee.
For instance, if we implement a system guaranteeing causal delivery, then in doing so, we would also be guaranteeing FIFO delivery, because FIFO delivery sits directly below Causal delivery in the hierarchy.
However, if we implemented a FIFO delivery system, we could make no guarantees about causal delivery.

Similarly, if the system implements Totally-Ordered Delivery, then this guarantee, in and of itself, cannot ensure either FIFO or Causal delivery.

Turning this argument around, we can also gain an understanding of what type of anomalies can occur.
For instance, if we have a FIFO anomaly, then this is also going to be a causal anomaly, but not necessarily a Totally-Ordered anomaly.

## Implementing Delivery Guarantees

### Implementing FIFO Delivery: Sequence Numbers

The rule here is that any process `P2` delivering messages from some other process `P1`, must do so in the order that `P1` sent those messages; which, due to variations in network latency, might well be different from the order in which those messages arrive at `P2`.

How then would we go about eliminating FIFO delivery anomalies?

One possibility is to use sequence numbers.
This is where all messages from a given sender are tagged with a sequence number and a sender id.
Each time a message is sent, the sender increments its sequence number.
On the receiver's side, all the messages from a given sender are added to a queue ordered by the sequence number.
When all the messages have arrived, the receiver can then deliver them in the correct order.

In this case, sequence numbers do not need to be unique across all the processes, because each message is also qualified with a sender id; therefore, it is the combination of the sender id and the sequence number that allows the receiver to discriminate who sent which message and in what order.

***Problems with Sequence Numbers***

What happens if a message is lost?
Consider the following sequence of events:

* `Alice` sends three messages to Bob: `m1`, `m2` and `m3`
* Bob receives and correctly delivers messages `m1` and `m2`
* For some reason, Bob never received message `m3`
* Unaware the message `m3` never arrived, `Alice` sends messages `m4` and `m5` which `Bob` receives
* However, because `Bob` is still waiting for the message with sequence number `3` to arrive, he will delay the delivery of any subsequent messages (by adding them to a queue)
* In this situation, `Bob` maye well end up waiting forever for the lost message to be delivered
 
![Naïve Sequence numbering](./img/L6%20Naive%20Seq%20Nos.png)

Consequently, in a network where message delivery is unreliable, a naïve sequence number strategy like this will break as soon as message delivery fails for some reason.

Strategies to mitigate these problems could include:

* Buffering out of sequence messages for a pre-determined period of time, hoping that the late message arrives either before the message buffer fills or the pre-determined timeout expires
* Processing out of sequence messages on the assumption that the intervening message is lost.
If this assumption turns out to be false and the message delivery was simply delayed, then the late message would have to be dropped

Neither of the above strategies are very good in that they tend to create more problems than they solve...

### Vacuous FIFO Delivery

Now consider this situation.
Are the conditions of FIFO delivery satisfied?

![Vacuous FIFO Delivery](./img/L6%20Vacuous%20FIFO%20Delivery.png)

Yes, but only in a vacuous sense.
Due to the fact that `Bob` drops all the messages he receives, zero messages are delivered; therefore, the conditions of FIFO delivery are vacuously satisfied.

### Implementing FIFO Delivery: Acknowledgments

In this approach, upon receipt of a message, every receiver must send a *"message received"* acknowledgment (such as `ack`) back to the sender.

So, when `Alice` sends a message to `Bob`, neither `Alice` nor `Bob` need concern themselves with sequence numbers.
However, this approach has several distinct drawbacks:

* Communication now becomes sequential.  `Alice` cannot send `m2` to `Bob` until she has received an `ack` from `Bob` that he has received `m1`
* Increases the volume of network traffic
* We are still dependent upon a network that can guarantee reliable message delivery

One way of making this approach to communication more efficient is to gather messages into batches, thus decreasing the granularity of communication.

## Using Vector Clocks to Prevent Causal Anomalies

Let’s look again at the ***Causal Anomaly*** situation:

![Causal Anomaly](./img/L3%20Causal%20Anomaly.png)

The problem here is that `Carol` delivers the message she receives from `Bob` out of causal order, thus resulting in confusion...

Here, we can use vector clocks to solve causal anomalies.

In the [previous lecture](./Lecture%205.md), we looked at using vector clocks to count both message-send and -receive events; but in order to ensure Causal Delivery, it turns out that we only need to count message send events.

![Ensure Casual Delivery 1](./img/L6%20Ensure%20Casual%20Delivery%201.png)

Before sending the message, `Alice` updates her vector clock to `[1,0,0]`

### What Should Bob Do?

`Bob` receives the message from `Alice`.

***Q:***&nbsp;&nbsp; Should he deliver it?  
***A:***&nbsp;&nbsp; Yes, he has no reason not to.

`Bob` delivers the message and discovers that it is not to his liking.
But, since `Bob` has the emotional maturity of an eight-year-old, he fails to realise that soap and water will work far better than trading insults; so, he resorts to telling the world what he thinks of `Alice`.

In delivering this message, `Bob` examines the vector clock of the incoming message and discovers that its less than his, but only by the counter in `Alice`'s position.
This is to be expected, since the message came from `Alice`.
He therefore uses the received vector clock to update his own vector clock, and then increments his position in the vector clock.

Since a broadcast message is treated as a single send event to multiple recipients, the same vector clock value of `[1,1,0]` is sent as part of the messages to both `Alice` and `Carol`.

![Ensure Casual Delivery 2](./img/L6%20Ensure%20Casual%20Delivery%202.png)

### What Should Alice Do?

The message now arrives at `Alice`.

***Q:***&nbsp;&nbsp; Should she deliver it?  
***A:***&nbsp;&nbsp; Yes, she has no reason not to.

`Alice`'s vector clock is `[1,0,0]` and the incoming vector clock on the message differs only by `1` in `Bob`'s position.
So, we can conclude that only one event has taken place since our last message send event, and that event happened in process `Bob` from whom we received this message.

### But What Should Carol Do?

`Bob`'s message also arrives at Carol with vector clock `[1,1,0]`, but earlier than `Alice`'s original message.

***Q:***&nbsp;&nbsp; Should she deliver it?  
***A:***&nbsp;&nbsp; No &mdash; look at the vector clock values!

The reason is that compared to `Carol`'s vector clock (which is still set to `[0,0,0]`), the vector clock on the incoming message is too big.

It’s fine for `Bob`'s position to be set to `1` because this is one bigger than `Carol`'s vector clock position for `Bob` and the message came from `Bob`.

But there's a `1` in `Alice`'s vector clock position.

***Q:***&nbsp;&nbsp; Hmmmm, that's odd.  Where did that come from?  
***A:***&nbsp;&nbsp; The value comes from the fact that this message is the response to some event that has taken place in `Alice`, but that ***Carol doesn't yet know about***.

In other words, as far as `Carol` is concerned, this is a ***message from the future*** that has arrived too early and must therefore be buffered.

Finally, `Alice`'s original `"Bob smells"` message arrives at `Carol`.
`Carol` now examines this message's vector clock and discovers that it has the expected value of `[1,0,0]`; therefore, it is fine to deliver this message first.

Once this out-of-sequence message has been delivered, the message waiting in the buffer can be delivered because `Carol` has now caught up with the event that took place in `Alice`.

`Carol` is no longer confused...

---

| Previous | Next
|---|---
| [Lecture 5](./Lecture%2005.md) | [Lecture 7](./Lecture%2007.md)

