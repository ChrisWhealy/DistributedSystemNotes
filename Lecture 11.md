# Distributed Systems Lecture 11

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 22nd, 2020 via [YouTube](https://www.youtube.com/watch?v=Rly9GBg14Zs)

## Implementing Reliable Delivery

### What Exactly is Reliable Delivery?

The definition we gave in the previous lecture for reliable delivery was the following

> Let `P1` be a process that sends a message `m` to process `P2`.  If neither `P1` nor `P2` crashes,<sup><b>*</b></sup> then `P2` eventually delivers message `m`

We also qualified this definition with the further criterion (indicated by the star) that says:

**\*** So, it appears there could be some circumstances under which we need to care about message loss, and some circumstances under which we don't.

The difference here depends on which failure model we want to implement.  If we are implementing the Crash Model, then the only kind of failures we will tolerate are processes crashing. So, under these circumstances, we will not handle message loss.  However, if we decide to implement the Omission Model, message loss must also be tolerated and therefore handled.

So, our definition of ***Reliable Delivery*** varies depending on which fault model we choose to implement.

From a practical perspective, the Omission Model is a far more useful approach for handling real-life situations since message loss is a real, everyday occurrence.

Remember also from the previous lecture that if we implement the Omission Model, we have also implemented the Crash Model.

![Fault Hierarchy 1](./img/L10%20Fault%20Hierarchy%201.png)

### But Can't We Have a Consistent Definition of "Reliable Delivery" 

Wouldn't it be better to have a consistent definition of "Reliable Delivery" that doesn't accumulate extra caveats depending upon the fault model?

Certainly - that would be something like this:

> If a correct process `P1` sends a message `m` to a correct process `P2` and not all messages are lost, then `P2` eventually delivers message `m`.

Ok, but haven't you just hidden the variablility of the earlier definition behind the abstract term ***correct process***?  Yes, that's true &mdash; but you did ask for a consistent definition!

The term ***correct process*** means different things in different fault models.  If we need to implement the Byzantine Fault Model, then a ***correct*** process is a non-malicious or non-arbitrary process; however, if we are implementing the Crash Model, then a ***correct*** process is simply one that doesn't crash.

So, as soon as you see the word ***correct*** in a definition like the one above, we should immediately determine which fault model being used, because this will then tell us what ***correct*** means in that particular context.


### How Do We Go About Implementing Reliable Delivery?

Going back to The Two Generals Problem discussed in the last lecture, one approach to help `Alice` and `Bob` launch a coordinated attack would be for `Alice` to keep sending the message `"Attack at dawn"` until she receives an `ack` from `Bob`.

This could be implemented using the following algorithm:

* `Alice` sends the message `"Attack at dawn"` to `Bob`, then places it into a send buffer that has a predetermined timeout period
* If an `ack` is received from `Bob` during the timeout period, communication was successful and `Alice` can delete the message from her send buffer
* If the timeout expires without receiving an `ack` from `Bob`, `Alice` resends the message

![Reliable delivery 1](./img/L11%20Reliable%20Delivery%201.png)

However, this style of implementation is not without its issues because it can never solve the Two Generals Problem:

* When `Alice` receives an `ack` from `Bob`, she can be sure that `Bob` got her message; however, `Bob` is still unsure that his `ack` was received
* Reliable delivery does not need to assume FIFO delivery, so copies of `Alice`'s original message could be received after `Bob` has already issued an `ack`.

Does it matter that `Bob` receives multiple copies of the same message?

In this situation, due to the nature of this particular message, no it doesn't.  This is partly because this particular strategy for increasing communication certainty requires `Alice` to send multiple messages until such time as she receives an `ack` from `Bob`; consequently, `Bob` will, mostly likely, receive multiple copies of the same message.

However, what if `Bob` was a Key/Value store and `Alice` wanted to modify the value of some variable?  Well, this depends entirely what `Alice`'s message contains.

![Reliable delivery 2](./img/L11%20Reliable%20Delivery%202.png)

If the message simply sets `x` to some absolute value, then this would not result in any data corruption.

![Reliable delivery 3](./img/L11%20Reliable%20Delivery%203.png)

However, if the message instructs the KeyStore to increment the value of `x`, then this will create significant problems if such a message were delivered more than once.

### Idempotency

The word ***idempotent*** comes from the Latin *idem* meaning "the same" and *potens* meaning "power".

In this context, the instruction contained in the message is said to be ***idempotent*** if (in mathematical terms):

`f(x) = f(f(x)) = f(f(f(x))) etc...`

In other words, an idempotent function only has an effect the first time it is applied to the data.  Thereafter, subsequent applications of that function to the same data have no further effect.

So, assigning a value to a variable is idempotent, but incrementing a variable is not.

Generally speaking, if we can work with idempotent operations, then our implementation of reliable delivery will be easier because we can be certain that nothing bad will happen to our data if, for some reason, the operation is applied more than once.

### How Many Times Should a Message be Delivered?

From the above discussion (and assuming that all communication happens within the asynchronous network model), we can say that reliable delivery therefore means that a message is delivered ***at least once***.

If we say that `del(m)` returns the count of the number of times message `m` has been delivered, then there are three possible options:

| Delivery Strategy | Delivery Count 
|---|---
| At least once | `1 ≤ del(m)` 
| At most once | `0 ≤ del(m) ≤ 1` 
| Exactly once | `del(m) = 1` 


Looking at the above table, it can be seen that since ***at most once*** delivery allows for a message to be delivered zero times, then this strategy can be implemented (at least vacuously) by not sending the message at all!  Doh!

But how about the ***exactly once*** strategy?

There does not appear to be any formal proof to demonstrate that this strategy is impossible to implement; however, it is certainly very hard to ensure.  Any time a system claims to implement ***exactly once*** delivery, the reality is that in the strictest sense, other assumptions have been made about the system that then give the appearance of exactly once delivery.  Such systems tend either to work with idempotent messages (which you can deliver as many times as you like anyway), or there is some sort of message de-duplication functionality at work.


## How Many Recipients Receive This Message?

Most of the message sending scenarios we've looked at so far are cases where one participant sends a message to exactly one other participant.

In [lecture 7](Lecture%207.md) we looked at an implementation of causal broadcast.  This is the situation in which ***all*** participants in the system receive the message (excluding of course the process that sent the message in the first place).  By means of vector clocks, we were able to ensure that a ***message from the future*** was not delivered too early.  

![Causal Broadcast](./img/L7%20Causal%20Broadcast%208.png)

Then there is the case that one participant sends a message to ***many***, but not all of the other participants in the system.  An example of this the Total Order anomaly we also saw in lecture 7.

![Total Order Anomaly](./img/L7%20TO%20Anomaly.png)

In this case `C1` sends messages to `R1` and `R2`, but not `C2`; likewise, `C2` sends messages to `R1` and `R2`, but not `C1`.

So, assuming reliable delivery, we have three different message sending strategies:

| Message Sending Strategy | Number of Recipients
|---|---
| Unicast | One
| Multicast | Many
| Broadcast | All


In this course we will not speak to much about implementing unicast messages; instead, we will simply assume that a unicast command exists as a primitive within each process.

In this manner, we could send broadcast or multicast messages simply by invoking the unicast primitive multiple times.

![Broadcast Implemented Using Unicast](./img/L11%20Broadcast%201.png)

Up until now, we have been drawing our Lamport diagrams with multiple messages coming from a single event in the sending process - and this is how it should be.  Conceptually, we need to treat broadcast messages as having exactly one point of origin.

However, under the hood, the mechanism for sending the actual messages could be multiple invocations of the unicast send primitive.  But this will only get us so far.  The problem is that even if we batch together all the message send commands in some transactional way, if we get halfway through sending this batch of messages and something goes wrong, what should you do about the messages sent so far - attempt to cancel or revoke them?

So, the reality is that we need a way to define reliable broadcast.


## Implementing Reliable Broadcast

Remembering the discussion of the term ***correct*** given in the section above, reliable broadcast can then be generically defined as:

> If the correct process delivers the broadcast message `m` then all correct processes deliver `m`.

***IMPORTANT ASSUMPTION***  
The discussion that follows assumes we are working within the Crash Model where we can pretend that message loss never happens.  Under these limited conditions, we only need to handle the case of processes crashing.

Let's say `Alice` sends a message to `Bob` and `Carol`.

Both `Bob` and `Carol` receive the message, and `Bob` delivers it correctly; however, `Carol` crashes before she can deliver that message.

![Reliable Broadcast 1](./img/L11%20Reliable%20Broadcast%201.png)

Has this violated the rules of reliable broadcast?

Actually, no it hasn't.  This is because since `Carol` crashed, she does not qualify as a ***correct*** process; therefore, its ok that she didn't deliver the message.

Now consider this next scenario: as we've mentioned earlier, under the hood, a broadcast message can be implemented as a sequence of unicast messages.  So, with this in mind, let's say that `Alice` wants to send a broadcast message to `Bob` and `Carol`; but as we now know, this will be implemented at two unicast messages that conceptually form a single send event.

So `Alice` sends the first unicast message to `Bob` who delivers it correctly.  But before `Alice` can send the second unicast message to `Carol`, she crashes.  This leaves both `Bob` and `Carol` running normally; however, `Bob` received the "broadcast" message but `Carol` did not.

![Reliable Broadcast 2](./img/L11%20Reliable%20Broadcast%202.png)

We could argue here that since `Alice` did not successfully send the message to ***all*** the participants, it is therefore not a true "broadcast" message. But this is unsatisfactory really because it sounds like we're trying to squeeze through a loophole in the definition of the word "broadcast".

In reality, `Alice` fully intended to send a message to ***all*** participants in the system; therefore, irrespective of the success or failure of this action, the intention was to send a ***broadcast*** message.  Therefore, this situation is in violation of the specification for a ***reliable broadcast***.

Notice that the definition of a reliable broadcast message speaks only about the correctness of the processes delivering that message; it says nothing about the correctness of the sending process.

Therefore, under this definition, if the correct process `Bob` received and delivered the message, then the correct process `Carol` should also receive and deliver this message.  Since `Carol` never received this message (for whatever reason), this is a violation of the rules of reliable broadcast.

So how can we implement reliable broadcast, if we know that halfway through the sequence of unicast sends, the sending process could crash?

Here's one outline of an algorithm for reliable broadcast:

* All processes keep a set of delivered messages in their local state
* When a process `P` wants to broadcast a message `m`:
    * It must unicast that message to all other processes (except itself)
    * `P` adds `m` to its set of delivered messages
* When `P1` receives a message `m` from `P2`:
    * If `m` is already in `P1`'s set of delivered messages, `P1` does nothing
    * Otherwise, `P1` unicasts `m` to everyone except itself and `P2`, and adds `m` to its set of delivered messages

Let's see this algorithm at work:

`Alice` sends a message to `Bob`, but then `Alice` immediately crashes.

However, since `Bob` has not seen that message before, he adds it to his set of delivered messages and unicasts it to `Alice` and `Carol` &mdash; but then `Bob` immediately crashes!

![Reliable Broadcast 3](./img/L11%20Reliable%20Broadcast%203.png)

Since `Carol` is the only ***correct*** process left running, she delivers the message.  However, since `Alice` and `Bob` have both crashed, they are excluded from our definition of a ***correct*** process, and so the rules of reliable broadcast remain satisfied.

Even with the optimization of not sending a known message back to the sender, this protocol still results in processes receiving duplicate messages.

![Reliable Broadcast 4](./img/L11%20Reliable%20Broadcast%204.png)

`Bob` and `Carol` each receive this message twice.

So, a further optimization can be that if a process has already delivered the received message, then simply do nothing.

## Fault Tolerance Often Involves Making Copies of Things

This is a fundamental concept that will be used a lot as we proceed through this course.

We can mitigate message loss by making copies of messages; but what else might we lose?

In addition to message sends and receives, there are also internal events within a process that record its changes of state (I.E. the changes that occur to a process' internal data).  How to we mitigate against data loss?  Again, by taking copies.

### Why Have Multiple Copies of Data?

There are several reasons

***Protection Against Data Loss***  
The state of a process at time `t` is determined by the complete set of events that have occurred up until that time.  Therefore, by knowing a process' event history we can reconstruct the state that process.  This then allows us to keep replicas of processes in different locations.  If one data centre goes down, then we still have all the data preserved in one or more other data centres.

***Response Time***  
Having your data in multiple data centres not only solves the issue of data loss, but it can also help with reducing response times since the data centre that is physically closest to you is the one most likely to give you the fastest response.

***Load Balancing***  
If you are experiencing a high volume of requests for the data, then having multiple copies of the data is a good way to distribute the processing load across multiple machines in multiple locations.


### Reliable Delivery

In the case of reliable delivery, we can tolerate message loss by sending multiple copies of the same message until at least one gets through.

### Reliable Broadcast

In the case of reliable broadcast, we can tolerate processes crashing by making copies of messages and then forwarding them.


## Question Received Via Chat

***Q:*** At the moment, we're only working within the Crash Model.  How would this work if we needed to work in the Omission Model?

***A:*** Remembering that fault models are arranged hierarchically, the algorithm used for reliable broadcast will act as the foundation for the algorithm used in the Omission Model.  Therefore, what we will need to do is extend the reliable broadcast algorithm with an algorithm for reliable delivery.





























