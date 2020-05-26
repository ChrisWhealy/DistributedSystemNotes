# Distributed Systems Lecture 17

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 8<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=-a1Ua1P9cNg)


| Previous | Next
|---|---
| [Lecture 16](./Lecture%2016.md) | [Lecture 18](./Lecture%2018.md)


## Amazon Dynamo

Please read the [Amazon Dynamo Paper](./papers/Dynamo.pdf) in preparation for the next lecture.  In the world of distributed systems, this is one of the most influential papers of the last 20 years, and also one of the most highly cited.

Before we discuss the paper however, we'll first look at some different, preparatory topics.

## Strong Consistency Between Replicas

So far, we have seen that strong consistency can be achieved using backup strategies such as Primary Backup Replication or Chain Replication.  We also saw that ultimately, strong consistency relies on consensus because in order to achieve fault tolerance, you need to have a coordinator, and under the surface, that coordinator is really a collection of processes acting as one.

The fundamental point here is this:

> If you want to implement a system that is strongly consist ***and*** you need it to tolerate node failure, then you ***must*** implement some sort of consensus mechanism.

The thing about strong consistency is that it is hard to enforce.  Consider the good old total-order anomaly example that we've seen so many times:

![Total-Order Anomaly](./img/L6%20Total%20Order%20Anomaly.png)

This anomaly is ***not*** a causal anomaly.  If it were a causal anomaly, then either the order of events leading up to a particular point in time would have become scrambled, or parts of that causal history are missing.  Such anomalies can be fixed by adding a vector clock to each message, thus allowing the delivering process to establish the correct message order.

But as we can see here, a total-order anomaly cannot be solved simply by adding vector clocks to the messages:

![Total-Order Anomaly with Vector Clocks](./img/L17%20TO%20Anomaly.png)

All the vector clocks have been incremented correctly, but if we compare these vector clock values between replica `R1` and replica `R2`, we cannot say that one is greater than the other.

So, using vector clocks alone, we are left in a situation where these two final events in `R1` and `R2` are causally independent making the total order of these messages is ***undecidable***.

So, what do we do here?

One option is to run a consensus algorithm, but as we have seen, such algorithms are expensive and might not terminate.  So, a better solution is to have a situation in which the order in which messages arrive at the replicas doesn't matter.

## Eventual Consistency

So, in what situation does the order of two updates not matter?  One situation is where two, unrelated values are being updated by two different clients.  

![Eventual Consistency](./img/L17%20Eventual%20Consistency.png)

***Q:***&nbsp;&nbsp; Given that the informal definition of strong consistency is that the clients cannot tell that the data has been replicated, is this form of consistency ***strong***?  
***A:***&nbsp;&nbsp;  No, this is not strong consistency because in between these writes, the clients can still do reads, and depending on the timing of the reads and writes and which replica they read from, this would allow them to discover differences in the state of `x` and `y`.

In spite of the fact that clients can discover differences in the data, these differences are short-lived and eventually consistency is achieved.  Whilst this is not perfect, it is a much better situation than having two replicas disagree with each other.


![Eventual Consistency](./img/Eventual%20Consistency.jpg)

So, informally, we say:

> ***Eventual Consistency***: Replicas eventually agree if clients stop submitting updates

***Q:***&nbsp;&nbsp; So, what sort of property is eventual consistency?  Is it a liveness property, or a safety property?  
***A:***&nbsp;&nbsp; It’s a liveness property because it cannot be violated in a finite execution.

Even in the case of the total-order anomaly we saw earlier where the two replicas disagreed on the value of `x`, we could still implement some message passing mechanism that would resolve this disagreement and achieve eventual consistency.

So eventual consistency is a very different consistency model than the other models we looked at so far such as:

* Strong consistency
* Causal consistency
* FIFO consistency
* etc...

All of these consistency models are ***safety*** properties (because you can violate them in a finite execution), whereas eventual consistency is a ***liveness*** property.

With eventual consistency, the clue is in the name ***eventual***.  This refers to the fact that consistency will be achieved, but only after  some unspecified period of time has elapsed &mdash; so, you could wait forever without violating this condition.

The term "*eventual consistency*" has been something of a buzzword in the last 15 years, and has often been mistakenly lumped together with strong, causal and FIFO consistency.  This is not the case because these are two different categories of property.

## Strong Convergence

Strong convergence is a safety property that is defined as:

> Replicas that have delivered the same ***set*** of updates have equivalent state

This is the situation illustrated by the diagram above.  Even though the updates to `x` and `y` arrived in a different order, the same set of updates arrived and both replicas eventually ended up holding the same state.

Thus, these replicas are said to be ***strongly convergent***.

***Strong Eventual Consistency*** = Eventual consistency + Strong convergence

Here we have combined the liveness property of eventual consistency with the safety property of strong convergence.


That's all very well, but doesn't this scenario seem rather unrealistic?  After all, we have assumed that client `C1` will only update key `x` and client `C2` will only update key `y`.  This doesn't really smell like a real-life situation.

So how can we allow strong convergence ***and*** allow both clients to update the same key?

Well, we could implement a consensus protocol, but that's a pretty heavyweight solution.  How could we do this without needing to go to these lengths?

So, here's an idea...  What if we kept both updates and instead of binding a single value to the key name, instead we bind a set of values:

![Strong Convergence 1](./img/L17%20Strong%20Convergence%201.png)

Since sets are unordered, the values held in each replica are considered equivalent.

The only issue now is that any client wanting to read `x` will get back a set of values rather than a single value.  It is now up to the client to perform conflict resolution and decide which value is authoritative.  The manner in which this conflict is resolved is application specific.

Conflict resolution aside, this situation conforms to the requirements of strong convergence because after the same set of updates have been applied by both replicas, they have converged to the same state. (The compromise here is on agreement about which value is the right one).

This situation is deeply important to Amazon as it concerns the replication of shopping carts.

Consider the situation where you want to add items to your shopping cart from two devices, your laptop and your phone.

![Amazon Shopping Cart 1](./img/L17%20Amazon%20Cart%201.png)

* From your laptop on the left, you add a book to your shopping cart.  
    This information is added to shopping cart 1 and then replicated over to shopping cart 2
    
* From your phone on the right, you also add a pair of jeans.  
    This information is added to shopping cart 2 and then replicated over to shopping cart 1

By the time these additions to the shopping cart have been processed, the state of your shopping cart in both replicas is the same.

But the key point here is that each replica holds ***two different versions*** of the shopping cart.  One version contains only the book, and the other only the jeans.

Now when a client performs a read of the shopping cart, it gets back both versions of the cart.

![Amazon Shopping Cart 2](./img/L17%20Amazon%20Cart%202.png)

So, what should the client do here to resolve this conflict?

Under the specific conditions of multiple devices adding items to the same shopping cart, it is not accurate to describe this situation as a ***conflict***.  In this specific situation, the actual state of the shopping cart is simply the union of the two sets, and this point is discussed in the Dynamo paper.

![Amazon Shopping Cart 3](./img/L17%20Amazon%20Cart%203.png)

However, simply forming the union of two versions of the data is not necessarily the correct approach in all situations.  

This is an example of a wider problem called *"Application-Specific Conflict Resolution"*, and this is a huge topic in its own right!

At this point, if you have questions related to how such conflicts can be resolved, that's not surprising &mdash; but unfortunately, we don't have time at the moment to look more deeply into this topic.

## Network Partitions

Another concept mentioned in the Dynamo Paper is that of ***Network Partitions***.

A network partition is where in a network of communicating computers, two subsets of computers can each talk to each other within their own subset, but the two subsets cannot communicate with each other.

![Network Partition 1](./img/L17%20Network%20Partition%201.png)

Rarely, it is also possible for one-way communication to happen across the partition boundary.

One significant issue for distributed systems is that network partitions can occur randomly, whilst communication between systems is taking place.

Fortunately, we have a way of talking about network partitions using the fault models we have already discussed.  Here, we can use the omission model and consider any message that attempts to cross the partition boundary as being lost.

> ***Aside***
> 
> Don't confuse the concept of network partitioning with data partitioning.
> 
> A network partition is a transient fault that disrupts or breaks communication between parts of a network &mdash; and this is generally considered to be a bad thing
> 
> Data partitioning (also known as ***sharding***) is where you have more data than will fit into one machine, so you need to split the storage across multiple machines.  This requires you to decide which data will live where.
> 
> Data partitioning is not intrinsically bad, but network partitions are.

## Availability

Dynamo claims to be a *"highly available"* system.  This is a relative term that describes the level of availability offered by the system to client requests.  Availability is best understood not as a binary property that is either present or absent, but as variable quality that sits somewhere on a sliding scale or a spectrum.

Perfect availability (which is not possible in reality) is the situation in which every request receives a response.  Ideally, we would also like to add that every request receives a ***fast*** response, but for the purposes of this discussion, we don't need to worry too much about response times.

Consider what would happen in a Primary Backup Replication scenario where the primary receives a write request, but then a network partition suddenly appears between the primary and its backups.

![Network Partition 2](./img/L17%20Network%20Partition%202.png)

The primary has now lost contact with its backups, so what should it do?

Should the primary simply acknowledge the write back to the client, then queue up the write to be sent to the backups as and when the network partition heals?

Well, yes, but this raises further issues...

1. Even though network partitions are generally short-lived events, you cannot say for certain how quickly they will heal
1. What if the primary crashes?  Now we have a whole load of issues:
    * If a backup now has to take over the role of the primary, data will have been lost because whichever backup takes over will be out of sync
    * We've already sent an acknowledgement to the client saying that their write was successful, yet when the backup takes over, the client's data will be missing.
    * The client will be able to detect the missing data, and will go away deeply saddened by all the trust issues this has created...

***Q:***&nbsp;&nbsp; So why doesn't the primary just wait for the partition to heal?  
***A:***&nbsp;&nbsp; Because this might take an arbitrarily large amount of time, or never happen.

The downside here is that network partitions are a fact of life, so this forces us to choose between what we hope will be the lesser of two evils:

* Should we wait for the partition to heal and risk waiting an arbitrarily long period of time?
* Should we update the primary and send an acknowledgement back to the client, but risk data loss if the primary crashes whilst the network partition exists?

It looks like we're caught between a rock and a hard place here.  Typically though, any system that implements strong consistency will prioritise consistency over availability.  In other words, to have the client experience a large response time is better than having to say to that client *"Sorry, we lost your data"*.

So, this means that the acknowledgement seen in the diagram above would never be sent unless the network partition has first been healed and communication with the backups restored.

If we now adjust our definition of availability to mean that every client receives a request *within some fixed amount of time*, even then we would be unable to make such a guarantee because we have no idea how long a network partition might take to heal.  There are however, some strategies for honouring a response time constraint.  We could:

* Inform the client that the update has been accepted, but no backup is available
* Inform that client that while the network partition exists, *"Updates are temporarily unavailable"*

Unfortunately, the last option here is somewhat vacuous because although we advertise that our system is highly available, what we really mean is that *"It's highly available, except when it isn't"*.  This is not really in the spirit of true high availability.

As we have already stated, any system that implements strong consistency will prioritise consistency over availability, which means that in the event of a hopefully short-lived network partition, the system will sit around and wait before responding to the client.  The highest priority here is to avoid data loss, and if required, this must be achieved at the expense of availability.

Amazon's Dynamo however prioritises things the other way around.  Dynamo prioritises availability at the expense of consistency.

The dilemma here is that you cannot guarantee both availability ***and*** consistency: the presence of failure in your system forces you to prioritise one over the other.  This is true regardless of the replication strategy you choose to use.

Consider a different situation now; here, a client can talk directly to two replicas, but a network partition has suddenly appeared between the two replicas.

The client sends an update to replica 1 changing the value of `x` from `4` to `5`.

![Consistency/Availability Trade-off 1](./img/L17%20Tradeoff%201.png)

Let's say that due to the sudden appearance of a network partition, the heartbeat between <code>R<sub>1</sub></code> and <code>R<sub>2</sub></code> fails, so <code>R<sub>2</sub></code> knows that it's lost contact with <code>R<sub>1</sub></code> and its data is now potentially out of sync.

![Consistency/Availability Trade-off 2](./img/L17%20Tradeoff%202.png)

<code>R<sub>2</sub></code> now receives a query for the value of `x`.  How should it respond?

## The Trade-off Between Availability and Consistency

<code>R<sub>2</sub></code> only has two, rather poor options:

1. Risk violating strong consistency by returning the potentially wrong answer of `x=4`, or
1. Risk violating availability by saying *“Hang on please, I need to phone a friend”* and hoping that the network partition heals very quickly

This problem is known as CAP: ***C***onsistency, ***A***vailability, ***P***artition Tolerance

You cannot achieve all three, so pick the two you need and then learn to deal with the problems created by the other one.

By now it should be no surprise to learn that this situation contains some subtleties that are often overlooked:

* The consistency being spoken of is specifically strong consistency
* The availability being spoken of is perfect availability

Even though you can't have all three of these qualities, it doesn't actually matter that much, because you can provide a system that has *"good enough"* availability.

For instance, Amazon does not claim that Dynamo offers *perfect* availability, only *high* availability and are prepared to offer a slightly weaker form of consistency.  Some other companies choose to balance these priorities slightly differently.

When designing a distributed system, you need to decide which of these qualities is of the greatest importance to you, and then build the system to provide you with the correct balance.  In Amazon's case as an online retailer, fast response times are of higher priority than the occasional lost item, so availability is prioritised over consistency.

So, systems are designed along a spectrum with availability at one end and consistency at the other.  Notice however, that partition tolerance is not shown on this spectrum.  This is because it is a nastier type of fault and is much harder to tolerate.

![Consistency/Availability Trade-off 3](./img/L17%20Tradeoff%203.png)

## Questions

***Q:***&nbsp;&nbsp; Where does MongoDB lie on this spectrum?  
***A:***&nbsp;&nbsp; More over to the availability end

However, systems like Dynamo are configurable in terms of the degree to which availability is prioritised over consistency (Dynamo has a configurable feature called "Quorum consistency")

When it was first published in 2007, one of the things that made the Dynamo paper so influential was that it contradicted the prevailing opinion that strong consistency must be given the highest priority.  Up until then, the majority of the research effort had assumed that the priority was to improve strong consistency and Byzantine fault tolerance.  However, Amazon realised if you emphasise strong consistency, you are then also forced to minimise the occurrence of network partitions; but in practice, this turns out to be extremely difficult simply because networks fail &mdash; get over it!

So, basically Amazon said *"Chasing after super high degrees of strong consistency is fool's errand because this also forces us to try to prevent the inevitable (network partitions).  So, let’s just accept that we'll get better results in the long term if we prioritise availability"* (or words to that effect...)

As soon as a highly successful online retailer said *"Not only do we not care about Byzantine Fault tolerance, we don't even care about strong consistency"*, people began to sit up and take notice.  This shift of priority became very influential for systems that came later because up until then, many companies had been over-engineering their approach to achieve high degrees of strong consistency.

---

| Previous | Next
|---|---
| [Lecture 16](./Lecture%2016.md) | [Lecture 18](./Lecture%2018.md)
