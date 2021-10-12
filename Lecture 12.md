# Distributed Systems Lecture 12

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 24<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=2dGJXEGTbGQ)


| Previous | Next
|---|---
| [Lecture 11](./Lecture%2011.md) | [Lecture 13](./Lecture%2013.md)


## Replication

Replication is the main strategy for mitigating loss:

* To avoid message loss, messages are copied and sent multiple times
* To avoid data loss, process state is copied

However, replication is used for more than just mitigating loss.

* ***Scalability/Load Balancing***  
    Handling sudden peaks in request volume or sustained high load
* ***Fault Tolerance***  
    Multiple, redundant instances of hardware/software
* ***Data Locality***  
    If you are physically close to your data, then you are likely to get a faster response to your requests.
    Hence, geographic distribution of data tends to provide better response times for users spread out around the world

However, what are the downsides of replication?

### What Are the Reasons Not to Do Replication?

Maintaining consistency of data across replicas is especially challenging when you consider that processes:

* Can crash due to software or hardware failure
* Are physically distant from each other
* Are continually changing state asynchronously
* Are connected by an unreliable communication network

But mitigation of these problems was the reason for wanting to do replication in the first place!
So, all the reasons for needing replication also make it hard to implement.

Another consideration is cost.

If a message is lost during transmission, you can simply resend it; so, the cost of transmission failure is paid only ***after*** the failure has occurred.
However, if a server crashes, you cannot avoid data loss unless you ***already*** have a replica server up and running.
So, to protect against hardware failure, you must incur the cost of running replica servers ***before*** anything has gone wrong.

## Total Order vs. Determinism

Looking back at our total order anomaly diagram, we can see that this problem raises the issue of maintaining data consistency across replicas.

![Total Order Anomaly](./img/L12%20TO%20Anomaly.png)

The problem here is that each replica holds a different value of `x`.

Let's look at the simplest case where, instead of replicating the data, we only have one copy.

Depending on the order in which messages arrive, we might end up with `x=2`

![Single Replica 1](./img/L12%20Single%20Replica%201.png)

Or, we might end up with `x=1`

![Single Replica 2](./img/L12%20Single%20Replica%202.png)

So simply reducing the number of replicas to one does not solve our problem.

A further problem is that neither of the situations described above can be considered wrong.
Both runs of the system are valid and correct, and neither violate Total-Order delivery.

Remember that if we want totally-ordered delivery:

> If a process delivers message `M1` followed by `M2`, then all processes delivering both `M1` and `M2` must deliver `M1` first followed by `M2`.

Process `R1` above is the only process delivering messages; therefore, it can never be in violation of the Totally-Ordered Delivery rule!
This gives process `R1` a lot of control over when to deliver messages.
In fact, process `R1` can decide for itself what *"Total Order"* actually means by controlling when it delivers messages.

In fact, `R1` could deliver messages in different orders across different runs and still not violate the safety property of Totally Ordered Delivery; however, it would be violating a different property called ***Determinism***

## Determinism

Determinism is a property that relates multiple runs of a system to each other.

![Determinism Violation](./img/L12%20Determinism%20Violation.png)

If process `R` delivers a set of messages in one order on its first run, but in a different order on the second run, then this is a violation of determinism.
I.E. The system is inherently unreliable because its behaviour is now indeterminate

> ***Totally-Ordered Delivery***  
> The property that describes consistent system behaviour over a single run
> 
> ***Determinism***  
> The property that describes consistent system behaviour over a multiple runs

BTW, any property relating to system behaviour across multiple runs is known as a ***hyperproperty***.

When there's only one replica, it will establish its own total order; however, when we have multiple replicas, we need each replica to behave with the same, consistent total order so that all the clients think they are dealing with a single system.

Here's an informal definition of what we want:

> A replicated storage system is ***strongly consistent*** if clients can't tell that it's replicated

As far as the clients are concerned, they should think that there is only one data storage system; even though, under the hood, there could be multiple replicas all working together.

Every strongly consistent replication protocol that we're going to discuss will implement a ***total order*** on events, but each will do so in different ways.

However, before we jump into these details, let's look at some of the ways that a client could tell that its working with multiple replicas.
To put this the other way around, let's look at the different ways in which replicas might disagree.

### Disagreements Between Replicas: Read Your Writes

Here's a possibility.

![Replica Disagreement 1](./img/L12%20Replica%20Disagreement%201.png)

The client wants to bind the value `5` to the name `x`, so it sends the message `x=5` to `R1`; but this value is not replicated across to `R2`.
So, when the client next queries the value of `x`, it gets back the unexpected result of `undefined` (in other words *"I don't know about any variable called `x`"*)

In this case, this is known as a ***Read Your Writes*** anomaly and is one of the most fundamental questions you should ask of any replicated storage system &mdash; that is, if I've just written a value, do I get the same value back when I perform a subsequent read?

There are some systems that still don't provide this!

### Disagreements Between Replicas: FIFO Consistency

> ***FIFO Consistency***  
> FIFO consistency is where writes done by a single process are seen by all processes in the order they were issued.

Here's a situation in which violation of this property results in replica disagreement.

![Replica Disagreement 2](./img/L12%20Replica%20Disagreement%202.png)

Let's says we have a banking system with two replicas `R1` and `R2`.
The client `C1` makes the following sequence of transactions against `R1`:

* Deposits \$50
* Receives an acknowledgement for the deposit
* Withdraws \$40
* Receives an acknowledgement for the withdrawal

Ok, that's fine.
`C1` can be very confident that her balance is \$10 &mdash; at least according to the records held in `R1`.


However, `R1` now sends out some synchronising messages, but `R2` delivers these messages in the wrong order.
After delivering the second message first, the client's balance is now incorrectly shown to be \$40 overdrawn.
If at this point in time, some automated balance checking process such as `C2` queries the client's balance, it will get the incorrect impression that this client has been spending too much money.

> ***Aside***  
> Wells Fargo Bank got in trouble for doing exactly this.
> They deliberately reordered transactions so that debits were processed before credits.
> This was done to reduce customer account balances to a minimum before calculating overdraft fees.
> Only then were the credits applied.

This is an example of a FIFO anomaly between replicas `R1` and `R2` and is a violation of FIFO consistency.

### Disagreements Between Replicas: Causal Consistency

> ***Causal Consistency***  
> Writes that are potentially causally related (I.E. related by the ***happens before*** relation `->`) must be seen by all processes in the same order

In this situation, all the events happen in chronological order such that the ***happens before*** relation is never violated.
However, replica `R2` is missing an event in its causal history and consequently gives the wrong answer to a query.

![Replica Disagreement 3](./img/L12%20Replica%20Disagreement%203.png)

The follow sequence of events happens:

* Client `C1` deposits $100 and receives an `ack`
* Client `C2` performs a balance enquiry against `R1` who accurately reports it as \$100
* `C2` then decides to withdraw \$50, but this request is processed by replica `R2`
* Since `R2` has no record of the event that deposited \$100, it accurately, but incorrectly declines the withdrawal on the grounds of there being insufficient funds

This problem is created by the fact that `R2` is missing an event in the causal history of the request to withdraw \$50.

***Q:***&nbsp;&nbsp; Why did `C2` makes its request against `R2` instead of `R1`?  
***A:***&nbsp;&nbsp; Well, in distributed systems, it is often the case that the client has no control over which replica serves its request.
Having said that, some distributed systems maintain consistency by forcing a client's requests to be served by the same replica.
However, decisions like this are taken by the distributed system and lie beyond the control of the client.

### Hierarchy of Consistency Guarantees

As with fault models, consistency guarantees can be arranged in a hierarchy.

![Consistency Hierarchy](./img/L12%20Consistency%20Hierarchy.png)

This is a very incomplete list - over 50 different types of consistency have been identified!
This is mentioned, not because we will need to learn all of these consistency guarantees in this course, but because different systems make different choices about which guarantees they feel it appropriate to make.

Higher up this hierarchy is not necessarily better.
There are cases when it makes good sense to offer only weaker consistency guarantees.

## Replication Strategies That Enforce Strong Consistency

Remember that we informally defined strong consistency as the case where a client cannot tell that the data has been replicated.

### Primary Backup Replication

Here the idea is pretty straight-forward and has been around since the 1970's.
We pick a system to act as the primary, and all the other systems act as backups.
This arrangement has the following advantages:

* It provides fault tolerance.
   If the primary fails, then one of the backups can immediately take over.
* Since the clients only ever talk to the primary, whatever total order is used by the primary is sufficient to maintain consistency.
   However, if we got rid of the division between primary and backup system and allowed any system to service client requests, then we would need to solve the harder problem of establishing a consistent total order across all these systems.

In this scenario, clients only ever interact with the primary.
When a client write message arrives at the primary, it is replicated to all the backup systems in parallel, each of which must send back an `ack`nowledge to the primary.

![Primary Backup Replication 1](./img/L12%20Primary%20Backup%20Replication%201.png)

When the primary receives `ack`s from all its replicas, it then delivers the message to itself, and finally sends an `ack` back to the client.
This is known as the ***Commit Point***.

![Primary Backup Replication 2](./img/L12%20Primary%20Backup%20Replication%202.png)

When the client wants to read a value, the answer need only come from the primary.
There is no need for reads to be broadcast to the replicas.

### Primary Backup Replication: Drawbacks

There are several drawbacks to primary backup replication:

* During a write operation, the fastest response time can be no less than the time taken for the slowest replica to send back its `ack`, plus the time taken for the primary to deliver the message to itself
* Under high load conditions, the primary becomes a bottleneck
* Having only one primary has at least two limitations:
    * Under high load situations, we cannot spin up more instances of the primary (horizontal scaling)
    * We cannot ease response time problems created by data locality

But could we do better?

Yes, we could redirect all reads to the backups and leave the primary to handle only writes.  

On the surface, this looks like a good solution, but upon closer examination, we discover that we could end up with some weird timing issues and potentially start reading ***data from the future***.

How so?

If a write is issued to the primary, that write is first broadcast to all the backups.
However, the backups only ever send their `ack`s back to the primary, not the client.
The primary does not deliver the message to itself until it has first received `ack`s from all its backups.
Only then does it send an `ack` back to the client.

This means that if we direct a read request to a backup for data that has just been updated, but before the `ack` has reached the client (via the primary), then potentially, we could be reading data from a state held in one of the backups that is ***ahead*** of the primary.
I.E. we will be reading ***data from the future***

But we can fix this problem by making a small change to the way `ack`s are handled.

Which leads us to...

### Chain Replication

In chain replication, the last backup to receive the write request does not send its `ack` back to the primary, but directly to the client.  

In Chain Replication, the process that we previously called the "Primary" is now called the "Head", and the process that acted as the last backup is now called the "Tail".
In between, there can be any number of processes that here, we will simply call "Backup".

The division of labour is now modified slightly:

* All write requests from clients are handled by the "Head" process
* The "Head" delivers the nmessage to itself, then passes the write instruction to the first backup in the chain
* Each backup delivers the message to itself, then passes the write request down the chain until it reaches the "Tail"
* When the "Tail" process completes message delivery, we know that since it is the last link in the chain, all the other replicas must ***already*** have completed their writes, so we can send an `ack` directly back to the client

![Chain Replication - Write](./img/L12%20Chain%20Replication%201.png)

This then means that should the client wish to make a subsequent read; it can direct that request to the backup from which it received the `ack`.

![Chain Replication - Read](./img/L12%20Chain%20Replication%202.png)

This is a relatively new strategy that was first published by Robbert van Renesse and Fred Schneider in a 2004 paper called ["Chain Replication for Supporting High Throughput And Availability"](./papers/chain_replication.pdf)

***Q:***&nbsp;&nbsp; But how can this be faster?
The response time experienced by the client will now be the ***sum*** of the times taken for each process to complete the message delivery and propagate the request through to the next link in the chain.
How can this then be described as *High Throughput*?

***A:***&nbsp;&nbsp; Well, to answer this question, we must first define what we mean by "Throughput".

> **Throughput**: The number of operations a system can perform per unit of time

The answer to this question also depends on the ratio of reads and writes we expect our system to handle.
For systems the perform mostly writes, then Primary Backup Replication will probably achieve higher throughput, but for systems that perform mostly reads, Chain Replication will probably achieve higher throughput.

### Chain Replication: Drawbacks

In Primary Backup replication, when the primary performs a write, the replication messages are broadcast (I.E. sent out in parallel) to all the backup processes.
Therefore, the longest wait time will be no longer than the time taken for the slowest backup to complete the message delivery and send out its `ack`.
So, no matter how many backups you have, the worst-case scenario will never be worse than the slowest individual backup process.

However, in Chain Replication, the length of the backup chain defines the overall response time for a write to complete.
As the chain length increases, so the overall response time increases because write replication propagates sequentially down the chain.

This then increases the system's ***write latency***.

What about read latency though?
Read latency for a Chain Replication system does not vary with chain length because all reads are directed to the tail.

Both of these strategies are commonly used; however, the decision as to which one will work best for you is governed primarily by the balance of reads and writes you expect your system to receive.

---

| Previous | Next
|---|---
| [Lecture 11](./Lecture%2011.md) | [Lecture 13](./Lecture%2013.md)

