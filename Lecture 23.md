# Distributed Systems Lecture 23

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on June  1<sup>st</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=tqCUDPWE0JE)

| Previous | Next
|---|---
| [Lecture 22](./Lecture%2022.md) |

## Original Plan for this Lecture

Originally, the plan for this lecture was for Lindsey to talk about her own research, but she changed her mind.  If you'd like to read/watch more about Lindsey's research, please use the following resources:

* [Abstractions for Expressive, Efficient Parallel and Distributed Computing](https://www.youtube.com/watch?v=4h7YBUXiCZE)
* [Lindsey's homepage at USCS](https://users.soe.ucsc.edu/~lkuper/)

## Recap of a Diagram Used by Chris Colohan

Let's go back to the illustration Chris Colohan used in his talk on [Blockchain Consensus](https://www.youtube.com/watch?v=m6qZY7_ingY) given to this class on May 27<sup>th</sup>, 2020.

He used a diagram that illustrates various strategies for implementing fault-tolerance.

![Fault Tolerance Hierarchy](./img/L23%20FT%20Hierarchy.png)

The point here is that everything in distributed systems is a trade-off.  In order to achieve goals such as low response times, high throughput, implementation simplicity and low running costs, you must accept the risk that if certain types of fault occur, then these will have a potentially significant impact on your system.  Depending on how significant the consequences of these faults are, you have to make an informed decision that balances the consequences of failure with the cost of protecting against such failure.

Since Paxos is the only strategy we've really talked about during this course, let's fill in a few of these gaps.

### 2-Phase Commit (2PC)

In the discussions we've had in this course, we have been talking about Primary Backup Replication and Chain Replication as the more basic forms of fault-tolerance.  These two techniques could be placed at the same location as 2-phase commit in this diagram.

2-phase commit is typically used in a database that operates with some number of replicas.  Here, everyone needs to agree on what action should be taken for the current logical unit of work (LUW).  There is typically some coordinating process that goes out to the other replicas and asks them the "go/no go" question of *"Do you want to commit or abort?"*

But why would one replica want to abort the LUW when the others are happy to commit?  Typically, this situation happens due to some hardware issue such as running out of disk space.

A key difference here between 2PC and a consensus algorithm is that the coordinator cannot accept a commit response from a majority of replicas, it must receive unanimous agreement from ***all*** the replicas before the commit can proceed.

If all the replicas respond with `commit`, then the coordinator will instruct all the replicas to commit, otherwise it instructs them to abort.

The point here is that just like Primary Backup Replication or Chain Replication, if you want fault-tolerance, then you must have a primary or coordinator process.  However, having only one coordinator process still does not provide fault-tolerance because that coordinator might crash.  Therefore, in order to implement fault-tolerance, you need a cluster of coordinator processes that can all agree on a course of action by means of some sort of consensus protocol.

Alternatively, you could use a protocol such as Paxos or RAFT in which there is no distinct "coordinator" process, but all the replicas coordinate directly with each other whilst performing at least one of the roles of proposer, acceptor or learner.

This diagram is good for illustrating the fact that 2-phase commit is typically not designed to be fault-tolerant: 2PC implementations typically have only one coordinator process.

The obvious question at this point then is this:

> Why not implement 2-phase commit ***and*** have a cluster of coordinator processes to provide fault-tolerance?

Well, this has already been done and is described in the first paper we're going to look at.

### Consensus on Transaction Commit

This is a [2006 paper](./papers/paxoscommit-tods2006.pdf) by Jim Gray and Leslie Lamport.

This paper describes a 2-phase commit system in which there is a cluster of coordinator processes that run a Paxos consensus algorithm between them on the commit/abort decision.

Here, a system is described in which `F` coordinator processes are allowed to fail, thus requiring the system to have a total of `2F + 1` coordinators up and running when it starts.

The original, single-process 2PC system described above is where `F` in the above equation equals `0`.  In other words, zero processes are permitted to fail because the system has started with only one coordinator in the first place.

One of the key points to realise here is that the concepts of consensus and replication are inextricably linked.  One of the most common reasons to implement a consensus algorithm is to perform replication.

How those replicas agree is less important.  You might choose to have a cluster of coordinator processes that tell the replicas when to commit or abort; or, you might choose to implement Paxos so the replicas can directly coordinate with each other.

### Practical Byzantine Fault-Tolerance (PBFT)

This is another [classic paper](./papers/pbft.pdf) published in 1999 by Miguel Castro and Barbara Liskov. Barbara Liskov has made some fundamental contributions to programming language design, especially in the area of object-oriented design.

PBFT is a replication technique that tolerates Byzantine faults, and in spite of having the word *"practical"* in the name, it is not used in practice that often.  

The distinguishing feature of this fault-tolerance behaviour is that if you have a system with `n` replicas, then `floor((n - 1) / 3)` of those replicas are allowed to "fail".  The world "fail" is put in quotation marks because what we mean here is really *"exhibit Byzantine behaviour"*.  By this, we mean that a process could:

* Genuinely crash
* Pretend to crash
* Disappear due to a network partition
* Act in a malicious way
* Some other behaviour not listed here...

It turns out that in practical terms, PBFT requires a minimum of 5 replicas in order for the system to tolerate a single failure.

Paxos, on the other hand, can work just fine if 1 out of 3 acceptors fail, because 2 is still a majority (and here, we're simply talking about crash failures and omission failures, not Byzantine behaviour).

Since protection against Byzantine behaviour is a stronger notion of fault-tolerance, then it makes sense that we require a minimum of 5 replicas for this strategy to work.

As we saw in the diagram above however, the more replicas you have, the more expensive and complex the strategy is to implement, and the slower it runs due to the higher number of replicas that must all agree before anything can get done.  So overall, a much bigger investment is needed to protect against a type of fault that might not happen very often.

This paper was very influential when it was released because previous strategies for protecting against Byzantine faults were much slower; unfortunately, it’s still not as fast as ***not*** protecting against Byzantine faults.

### Blockchain "Consensus"

Of all the available fault-tolerance models, this one is the most expensive to run.  This is mainly because you need some seriously powerful number-crunching hardware to perform blockchain mining.  This in turn gives you very large electricity bill and increases your impact or the environment...

The term consensus is in quotes here because in strategies such as Paxos and RAFT, there is a very definite point in the algorithm where you know that consensus has been reached, and then there is no going back.  With Blockchain "consensus", the algorithm never reaches a specific milestone; instead, you only have a probabilistic guarantee that consensus is likely.


### Where Does This Leave Us?

If you can get away with not needing strong consistency between replicas, then that will be a good choice because it will save you a large amount of time, energy and money both during the implementation and at runtime.

Overall, go for the weakest safety property you can reasonably tolerate.  Hence the suggestion to use strong convergence instead of strong consistency, or the development of techniques such as Conflict-Free Replicated Datatypes (CRDTs).

## Discovering Who Invented Vector Clocks

It is not clear who first invented vector clocks, and in many respects, attribution of this concept to a single person or group is not of much consequence; however, it is interesting to trace the development of this idea.

Firstly, we can place a lower bound on the date by referring to the first paper to treat distributed systems as a science.  This paper was written by Leslie Lamport in 1978 and was called [Time, Clocks and the Ordering of Events in Distributed Systems](./papers/TCOEDS.pdf).  In it, the concept of the *"happens before"* relation was properly defined as was the notion of a Logical or Lamport Clock.

A couple of interesting things to notice about the space-time diagrams (now known as "Lamport Diagrams") shown in this paper are:

1. Stylistically, they bear a passing resemblance to Feynman Diagrams (maybe there is an influence from physics because the final section of this paper on physical clocks seems only to make sense to physicists; and even then, that proposition is not entirely sound...)
1. He has time going upwards; which does tend to make them harder to read

Later, these diagrams have been drawn to show time going left, right or down.  Lamport however, is one of the few people to draw these diagrams with time going up.

### Concepts Introduced in Lamport's First Paper

* Logical, or Lamport Clocks
* The *"Happens Before"* relation denoted by the arrow `->` syntax
* The idea of the *"clock condition"* that can test whether the happens before relation is satisfied for a pair of events.  
    For events `a` and `b`, if `a -> b` then `C(a) < C(b)` where `C(a)` and `C(b)` are the respective *"clock conditions"* of events `a` and `b` - in other words, the value of the Lamport Clock associated with each event.
* Initial proposition for turning the partial order implemented by Lamport Clocks into a total order.  This idea, however, requires that if two events have the same clock value, then an arbitrary algorithm is used to decide the outcome; for instance: events happening on process `P1` might always be assumed to have happened before events on process `P2`.
* Distributed Mutual Exclusion Algorithm.  This controls how to manage exclusive access to a shared resource for a group of processes. However, no one actually uses this algorithm for the simple reason that it's not fault-tolerant.
* The observation that distributed processes simulate the execution of State Machines

### The "Holy Grail" Paper

Although the idea of vector clocks was developed by several researchers independently, and certainly before the publication of this paper, the ["Holy Grail" paper](./papers/holygrail.pdf) is worth reading because it describes how to detect causal relationships in distributed systems.

It covers topics such as:

* How in certain circumstances, vector clocks provide a better mechanism than Lamport Clocks for determining causal relationships
* Are vector clocks even the best solution?

Footnote 3 of this paper makes an interesting comment:

> Actually, the concept of vector time cannot be attributed to a single person. Several authors “re-invented” time vectors for their purposes, with different motivation, and often without knowing of each other. To the best of our knowledge, the first applications of “dependency tracking” vectors [70] appeared in the early 80’s in the field of distributed database management [21, 74]. In [17] and [44], however, vector time is introduced as a generalization of Lamport’s logical time, and its mathematical structure and its general properties are analysed.

At the end of this footnote, two references are given to a paper by Colin Fidge [17] and a paper by Friedemann Mattern [44]

### Colin Fidge's Paper: "Timestamps in Message-Passing Systems That Preserve the Partial Ordering"

[This paper](./papers/fidge88timestamps.pdf) was published by Colin Fidge in 1988 and without calling them *"vector clocks"* he invented the same concept by saying:

> Rather than a single integer value, timestamps are represented as an array
> 
> <code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<i>[C<sub>1</sub>,C<sub>2</sub>,&hellip;,C<sub>n</sub>]</i>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code>
> 
> with an integer clock value for every process in the network.



### Friedemann Mattern's Paper: "Virtual Time and Global States in Distributed Systems"

[This paper](./papers/VirtTime_GlobState.pdf) was published by Friedemann Mattern also in 1988 where, on the second page, he states:

> In this paper, we aim at improving Lamport's virtual time concept. We argue that a linearly ordered structure of time is not always adequate for distributed systems and that a ***partially ordered system of vectors*** forming a lattice structure is a natural representation of time in a distributed system

There's the use of vectors again to hold a set of individual clock values.

Then in section 3, Mattern talks about *Consistent Cuts* that we have already looked at in the context of snapshots and the Chandy-Lamport Algorithm.  The interesting thing here is that although the Chandy-Lamport algorithm predates Mattern's paper by a few years, at the time Chandy and Lamport devised their snapshot algorithm, vector clocks had not yet been invented.

Here, Mattern now ties together (apparently for the first time), the use of consistent cuts with the use of clock values held in vectors.

### Frank Schmuck's 1988 PhD Paper

In 1988, a German PhD student at Cornell by the name of Frank Schmuck<sup id="a1">[1](#f1)</sup> published his thesis on ["The Use of Efficient Broadcast protocols in Distributed Systems"](./papers/Frank%20Schmuck%20PhD%20Paper.pdf)

On page 53, Schmuck states:

> In [Lam78] Lamport introduced *logical timestamps*, integers assigned to each event in such a way that if all events are ordered by their timestamp this order is consistent with "`->`". We can generalize this idea to timestamps which are ***vectors of integers*** [Sch85].<sup>2</sup>  Such timestamps are useful for keeping track of the partial order of events as the system executes.
>
> A timestamp *`t`* for an event <code>e = event<sub>R</sub>⟨i,j⟩ ∈ R</code> is a vector of *`n`* integers with the following meaning:
> 
> <code>t<sub>e</sub>[k]= || {event<sub>R</sub>(k,l) ∈ <sub>k</sub> | event<sub>R</sub>(k,Z) → e} ||</code>
> 
> i.e., <code>t<sub>e</sub>[k]</code> is the number of events at *`k`* that precede *`e`* in the partial order.

There's that phrase again *"vectors of integers"* and a citation to his own, unpublished manuscript of 1985.  However, footnote 2 says the following:

> <sup>2</sup> The idea of vector timestamps was developed independently by Ladin and Liskov [LL86].

So, let's chase after the paper referenced as LL86.

### Ladin & Liskov's Paper: "Highly-Available Distributed Services and Fault-Tolerant Distributed Garbage Collection"

[This 1986 paper](./papers/Ladin%20and%20Liskov.pdf) by Rivka Ladin and Barbara Liskov did not use the precise name *"vector clock"*; however, it did use the name *multipart timestamps*:

> We solve this problem by using multipart timestamps, where there is one part for each replica. Thus, if there are `n` replicas, a timestamp `t` is
> 
> <code>t = &lt;t<sub>1</sub>, &hellip;, t<sub>n</sub>&gt;</code>
> 
> where each part is a positive integer. Since there will typically be a small number of replicas (e.g., 3 to 7), using such a timestamp is practical.

### Skipping Forward to 1991 and Causal Broadcast

Frank Schmuck's doctoral supervisor was Ken Birman, and in 1991, he published a paper called [Lightweight Causal and Atomic Group Multicast](./papers/birman91multicast.pdf).  In section 4.3 called *"Vector Time"*, they now start to talk about vector clocks, and their description is exactly what we discussed in [lecture 5](./Lecture%205.md#vector-clocks).  This discussion of vector clocks then forms the foundation of the paper's main subject, which is Causal Broadcast.

The key part of the Causal Broadcast discussion then states in section 5.1 that after a message has been received, its delivery is delayed until various conditions have been fulfilled concerning the vector clock values in the message and the vector clock values held in the receiving process.

In class we spoke of the rule that when a message is delivered, the vector clock value in the sender's position should be incremented; but here in Birman's paper, he says effectively *"just merge the sender and receiver's clocks using a pointwise maximum"*.  Although this sounds like a discrepancy, it turns out that these two approaches are functionally equivalent.

### The Chandy-Lamport Paper

In 1985, K. Mani Chandy and Leslie Lamport published a paper that we've already looked at in [Lecture 8](./Lecture%208.md) called ["Distributed Snapshots: Determining Global States in a Distributed System"](./papers/chandy.pdf).

There is a really nice quote in the introduction that helps summarise what they are trying to achieve. They describe their algorithm as a *state-detection* algorithm that allows them to understand the global state of a distributed system:

> The state-detection algorithm plays the role of a group of photographers observing a panoramic, dynamic scene, such as a sky filled with migrating birds-
a scene so vast that it cannot be captured by a single photograph. The photographers must take several snapshots and piece the snapshots together to form a picture of the overall scene. The snapshots cannot all be taken at precisely the same instant because of synchronization problems. Furthermore, the photographers should not disturb the process that is being photographed; for instance, they cannot get all the birds in the heavens to remain motionless while the photographs are taken. Yet, the composite picture should be meaningful. The problem before us is to define “meaningful” and then to determine how the photographs should be taken.

Using the analogy of photographing a vast number of migrating birds, this paragraph encapsulates several key concepts:

1. The overall view of the system is too big to be captured in a single snapshot event, so multiple snapshots must be taken and pieced together
1. For practical reasons of synchronization, these multiple snapshots cannot all be taken at the precisely the same point in time
1. The act of taking a snapshot must not disturb the running system

## Who Was the First to Come Up with Primary Backup Replication?

### Alsberg and Day

The paper that tends to be cited the most here is called ["A Principle for Resilient Sharing of Distributed Resources"](./papers/Alsberg%20and%20Day.pdf) by Peter Alsberg and John Day, first published in October 1976.  However, the methodology described in the paper does not look very much like what we now call primary backup; nonetheless, Alsberg and Day did produce a working system that operated in the packet switched networks of the day, where data packets sent over the network really could get lost.

When reading these old papers that were written when the nearest thing to the "internet" was [ARPANET](https://en.wikipedia.org/wiki/ARPANET) or [CYCLADES](https://en.wikipedia.org/wiki/CYCLADES), you have to admire the ingenuity of these people and their ability to solve hard problems in what we would now describe as a very hostile environment.  Many (though not all) of the assumptions they made 40+ years ago are still valid today.


### Reliable, Autonomic Distributed Store (RADOS)

This is part of the now Open-Source system called [Ceph](https://en.wikipedia.org/wiki/Ceph_(software)).

Moving closer to home, the 2007 paper called ["RADOS: A Scalable, Reliable Storage Service for Petabyte-scale Storage Clusters"](./papers/rados.pdf) contains a nice diagram that shows a hybrid of Primary Backup Replication and Chain Replication, that they call "splay replication".

![RADOS Paper Figure 2](./img/L23%20Rados%20Fig%202.png)

This has the features of Primary Backup (also known as Primary Copy) in that the writes are received at the primary node and broadcast in parallel to the other replicas.  However, the principles of Chain Replication are now used insomuch as the last replica in the chain is responsible for sending an `ack` back to the client, and all reads are directed to the tail replica.

So, the interesting thing here is that even in 2007, people were still working towards improving strongly consistent replication strategies.

## Consensus

What is interesting about both these papers (and many others not mentioned here) is that anytime someone sets out to talk about replication, they end up needing to talk about consensus.  However, rather than using a general purpose consensus algorithm such as Paxos, they always talk of a consensus algorithm that has been specially tailored for their particular use case.

Paxos, on the other hand, is a general purpose, one-size-fits-all consensus algorithm.  However, the general problem with the one-size-fits-all approach is that everyone ends up wearing size XXXL.  This is the same with Paxos: in order to make it efficient in a replication scenario, you need to implement optimisations such as multi-Paxos.  With the following algorithms however, they simply assume that you want to do replication and provide you with a consensus algorithm already customised for that use-case.

See the ["Vive la Différence"](./papers/Paxos%20vs%20VSR%20vs%20ZAB.pdf) paper for a comparison of these various strategies.  The great thing about this paper is that at the top of page 6, there is a table of equivalent terminology across the different replication protocols.  What this demonstrates is that while different terms are used, the same basic concepts always pop up.

![Equivalent Terminology Across Different Replication Protocols](./img/L23%20Equivalent%20Terms.png)

### The RAFT Paper

The [RAFT paper](./papers/raft.pdf) came out in 2014 and emphasises understandability over Paxos's complexity.  They have a user study in which they compare users' impressions of how easy RAFT is to use compared to Paxos.

This system is very well documented on their website <https://raft.github.io/>

The authors cite that the RAFT consensus algorithm is similar to the earlier consensus protocol called Viewstamped Replication.

### Viewstamped Replication

The [Viewstamped Replication](./papers/VS520Replication.pdf) paper immediately places us in situation where in one breath we are calling it a replication strategy, and then in the next, we are calling it a consensus algorithm.

Aren't these different things?

Well, yes - in a no kind of way.

Many of the replication strategies that we’ve looked can only work because under the surface, they implement a consensus protocol.  So generally speaking, replication needs consensus, and consensus is used primarily to facilitate replication.


### Are Paxos and RAFT So Different?

Very recently in April 2020, a [comparison paper](./papers/Paxos%20vs%20RAFT.pdf) was published by Heidi Howard and Richard Martier in which a simplified version of Paxos was described using the terminology of RAFT.  The paper concludes that Paxos and RAFT contain a lot of similarities, but RAFT has become more popular because it is easier to understand.  For example, RAFT is very easy to implement from its paper, whereas it is very challenging to implement Paxos from its paper.


---

| Previous | Next
|---|---
| [Lecture 22](./Lecture%2022.md) |


---

***Endnotes***

<b id="f1">1</b>  Stop giggling... Just in case your mind has fixated on the [Yiddish meaning](https://en.wikipedia.org/wiki/Schmuck_(pejorative)) of this word, ["Schmuck"](https://dict.leo.org/german-english/Schmuck) is the German word for "jewellery" or "decoration" or "ornament"...

[↩](#a1)

