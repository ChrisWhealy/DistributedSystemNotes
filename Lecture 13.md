# Distributed Systems Lecture 13

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 27<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=5oCUmo9PKaw)

| Previous | Next
|---|---
| [Lecture 12](./Lecture%2012.md) | [Lecture 14](./Lecture%2014.md)


## Primary Backup (P/B) Replication

In P/B Replication, the clients only ever talk to the primary node `P`.  Any time `P` receives a write request, that request is broadcast to all the backup nodes, which independently send their `ack`s back to the primary.

When the primary has received `ack`s from all its backups, it then delivers the write to itself and sends an `ack` back to the client.  This point in time is known as the ***commit point***.

The write latency time experienced by the client is the sum of the times taken to complete each of the following four steps (imagine we have some function `rt_time` that can measure the response time between two nodes):

<code>rt\_time(C, P) + rt\_time(P, B<sub>slowest</sub>) + rt\_time(B<sub>slowest</sub>, P) + rt\_time(P, C)</code>

Irrespective of the number of backups in this system, all write requests are completed in these four steps.

![Primary Backup Replication - Writes](./img/L12%20Primary%20Backup%20Replication%201.png)

Read requests are handled directly by the primary.

![Primary Backup Replication - Reads](./img/L12%20Primary%20Backup%20Replication%202.png)


The read latency time is the sum of the time taken to complete the two steps:

`rt_time(C, P) + rt_time(P, C)`

### Primary Backup Replication: Drawbacks

There can only be one primary node; thus, it must handle all the workload.

In addition to the primary becoming a bottleneck, this arrangement does not allow for any horizontal scalability.

## Chain Replication

Chain Replication was developed to alleviate some of the drawbacks of Primary Backup Replication.

![Chain Replication - Write](./img/L12%20Chain%20Replication%201.png)

Here, the write latency time grows linearly with the number of backups and is calculated as the sum of the time taken to complete the `3 + n` steps (where `n` is the number of intermediate backups between the head and the tail):

<code>rt\_time(C, H) + rt\_time(H, B<sub>1</sub>) + &hellip; + rt\_time(B<sub>n</sub>, T) + rt\_time(T, C)</code>

So, if you have a large number of backups, the client could experience a higher latency time for each write requests.

The point however of Chain Replication is to improve read throughput by redirecting all reads to the tail.

![Chain Replication - Read](./img/L12%20Chain%20Replication%202.png)

Now the read latency time is simply the sum of the time taken to complete the two steps:

`rt_time(C, T) + rt_time(T, C)`

Here are some figures from the [Chain Replication paper](./papers/chain_replication.pdf) by Renesse and Schneider.  Looking at Figure 4 at the top of page 8

![Chain Replication Paper - Figure 4](./img/L13%20Chain%20Replication%20Paper%20Fig%204.png)

These graphs compare the request throughput times of three different backup strategies (where `t` represents the number of replicas in the system):

* ***Weak Replication***  
    Client requests can be served by any replica in the system.  
    Indicated by the solid line with `+` signs
* **Chain Replication**  
    Client write requests are always served the head, read requests are always served the tail.  
    Indicated by the dashed line with `x` signs
* ***Primary Backup Replication***  
    All client requests are served the primary.  
    Indicated by the dotted line with `*` signs

As you can see, Weak Replication offers the highest throughput because any client can talk to any replica.  So, this is good illustration of how throughput can be improved simply by throwing more resources at the problem. However, it must also be understood that Weak Replication cannot offer the same strong consistency guarantees as either Primary Backup or Chain Replication.

Weak Replication therefore is only valuable in situations where access to the data is *"read mostly"*, and you're not overly concerned if different replicas occasionally give different answers to the same read request.

Comparing the Chain and P/B Replication curves, notice that if none of the requests are updates, then their performance is identical. The same is true when the update percentage starts to exceed about 40%.

However, look at the Chain Replication curve.

Instead of descending in a gradually flattening curve, there is a hump at around the 10-15% mark.  This is where the benefits of Chain Replication can be seen.

By why should this improvement be seen at this particular ratio of writes to reads?

The answer here lies in understanding how the workload is distributed between the head and tail processes in Chain Replication.  According to the research done by Renesse and Schneider, their experiments showed that when 10-15% of the requests are writes, then this produces the best throughput &mdash; presumably because the workload has now been distributed evenly between the head and tail processes.

It turns out that in practice, this ratio of writes to reads is quite representative of many distributed systems that are *"out there in the wild"*.


## Dealing with Failure

If the primary process in a P/B Replication system fails, who is responsible for informing the clients that one of the backups has now taken on the role of primary?

Well, clients always need to know who to contact in the first place, and this role could be performed by some sort of coordinator acting as a communication proxy.

In the P/B Replication system, the coordinator must at least ensure that:

* Each client knows who is acting as the primary
* Each replica knows who the head is

In Chain Replication, coordination is slightly more involved in that:

* Each client must know who is acting as the head ***and*** who is acting as the tail
* Each replica needs to know who the next replica is in the chain
* Everyone must agree on this order!

### Coordinator Process

So, in both situations, it is necessary to have some sort of internal coordinating process whose job it is to know who all the replicas are, and what role they are playing at any given time.

> ***Assumption***  
> This discussion makes the following assumptions:
> 
> * Not all the processes in our system will crash.  
>    For a system containing `n` processes, we are relying on the fact that no more than `n-1` processes will ever crash (Ha ha!!)
> * The coordinator process is able to detect when a process crashes.
>
> However, we have not discussed how such assumptions could possibly be true because the term *"crash"* could mean a variety of things: perhaps software execution has terminated, or execution continues but the process simply stops responding to messages, or responds very slowly...
> 
> Failure detection is a deep topic in itself that we cannot venture into at the moment; suffice it to say, that in an asynchronous distributed system, perfect failure detection is impossible.

The coordinator must perform at least the following roles:

* It must know about all the replicas in the system
* It must inform each replica of the role it is currently playing
* It must monitor these replicas for failure
* Should a replica fail, the coordinator must reconfigure the remaining replicas such that the overall system keeps running
* It must inform the clients which replica(s) will service their requests:
    * In the case of P/R Replication, the coordinator must inform the clients which replica acts as the primary 
    * In the case of Chain Replication, the coordinator must inform the clients which replica acts as the head and which acts as the tail 

#### Coordinator Role in P/B Replication

In the event of failure in a P/B Replication system, the coordinator must keep the system running by:

* Nominating one of the backups to act as the new primary
* Informing all clients to direct their requests to the new primary
* Possibly starting a new replica to ensure that the current system workload can be handled and configuring that replica to act as a backup
* etc...

#### Coordinator Role in Chain Replication

The coordinator must perform a similar set of tasks if failure occurs in a Chain Replication system.  If we assume that the head process fails, then the coordinator must keep the system running by:

* Nominating the head's successor to act as the new head
* Informing all clients to direct their write requests to the new head
* Possibly starting a new replica to ensure that the current system workload can be handled and reconfiguring the chain to include this new backup
* etc...

Similarly, if the tail process fails, the coordinator makes the tail process' predecessor the new tail and informs the client of the change.

### What if the Coordinator Fails?

If we go to all the trouble of implementing a system that replicates data across some number of backups, but uses a single coordinator process to manage those replicas, then in one sense, we've actually taken a big backwards step because we've introduced a new *"single point of failure"* into our supposedly fault tolerant system.

So, what steps can we take to be more tolerant of coordinator failure&hellip;

* Simply spin up some replicas of the coordinator?  And should we do this in just one data centre, or across multiple data centres?
* But then how do you keep the coordinators coordinated?
* Do you have a coordinator coordinator process?  If so, who coordinates the coordinator coordinator process?

This quickly leads either to an infinite regression of coordinators, or another [Monty Python sketch](./img/very_silly.png)... (Spam! spam! spam! spam!)

This question then leads us very nicely into the next topic of ***Consensus*** &mdash; but we won't start that now.

It is amusing to notice that in Renesse and Schneider's paper, one of the first things they state is *"We assume the coordinator doesn't fail!"* which they then admit is an unrealistic assumption.  They then go on to describe how in their tests, they had a set of coordinator processes that were able to behave as a single process by running a consensus protocol between them.

It is sobering to realise that if we wish to implement both strong consistency between replicas ***and*** fault tolerance (which was the problem we wanted to avoid in the first place), then ultimately, we are forced to rely upon some form of consensus protocol.

But consensus is both ***hard*** and ***expensive*** to implement.  This difficulty might then become a factor in deciding ***not*** to implement strong consistency.  Now it looks very appealing to say *"If we can get away with a weaker form of consistency such as Causal Consistency, then shouldn't we look at this option?"*

That said, there are times when consensus really is vitally important.

---

| Previous | Next
|---|---
| [Lecture 12](./Lecture%2012.md) | [Lecture 14](./Lecture%2014.md)


