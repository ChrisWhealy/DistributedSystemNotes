# Distributed Systems Lecture 14

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 1st, 2020 via [YouTube](https://www.youtube.com/watch?v=WgYwBLStCbs)


## Recap: Strongly Consistent Replication Protocols

These all work by establishing some sort of total order on operations

* ***Primary Backup (P/B) Replication***

    The write path is one where the client talks to the primary, then the primary broadcasts that write to all the backups.  Each backup then acknowledges the primary and when all the acknowledgments have been received, the primary delivers the write to itself and acknowledges the client.  The point at which the primary delivers the message to itself is known as the *Commit Point*.
    
    ![Primary Backup Replication - Write Path](./img/L12%20Primary%20Backup%20Replication%201.png)
    
    The read path is where the client talks directly to the primary
    
    ![Primary Backup Replication - Read Path](./img/L12%20Primary%20Backup%20Replication%202.png)

    If a backup crashes, it can simply be replaced by another backup; however, if the primary crashes, although a backup can take over, we must be careful to ensure that the state of the backup is not ahead of the state of the primary. In other words, the backup we are about to promote to the role of primary must not be in the process of performing writes that have not yet been acknowledged back to the client (via the now failed primary).  If this is not ensured, then promoting that backup to the role of primary will cause ***writes from the future*** to appear in the new primary, thus potentially confusing the clients.
    
    In order to manage this, an internal coordinator process is required that holds state about what the other processes are doing.  For instance, the coordinator must know:
    
    * Which nodes are up or down at any given time and what role they are playing
    * Monitoring all the nodes for failure (perhaps by some sort of ping or heartbeat mechanism)
    * Reassigning the role of a node in the event that some other node fails
    * Communicating with the clients in the event that a new node has taken over the role of primary

* ***Chain Replication***

    The write path is one where the client talks to the head of the chain.  The head then propagates the write request sequentially down the chain until it arrives at the tail node.  The *commit point* occurs when the tail node delivers the message to itself and send an `ack` back to the client.

    ![Chain Replication - Write Path](./img/L14%20Chain%20Replication%201.png)
    
    The read path is where the clients talk directly to the tail node.

    ![Chain Replication - Read Path](./img/L14%20Chain%20Replication%202.png)
    
    Similarly, in Chain Replication there also needs to be an internal coordinator process that performs all the tasks needed in P/B Replication, but additionally:
    
    * Knows the order of nodes in the chain
    * Can reconfigure the chain in the event of node failure
    * Informs the clients which node acts as the head and which node acts as the tail
    
* ***Coordinators - A Single Point of Failure?***

    The main reason for inventing these replication strategies is to mitigate the effects of process failure.  Redundant copies of a process are created so that in the event of failure, one of the copies can immediately step into the gap.  That's all very well, but without there being some coordinator process to monitor the processes doing the actual work, we're no closer to protecting ourselves against failure than when we started.
    
    The problem here is that the coordinator process is exactly that &mdash; a process &mdash; which means it’s just a liable to failure as any other process. So, if the coordinator fails, we're dead and everything falls apart.
    
    On its own, a single coordinator process is little more than a single point of failure; so, who monitors the coordinator?
    
    Well, we could have a coordinator coordinator; but then who monitors the coordinator coordinator? Another coordinator for the coordinator coordinator?  Hmmmm, this situation will rapidly turn into either an infinite regression of coordinator processes, or a [Monty Python sketch](./img/very_silly.png)...
    
    What we tend to do here is implement a replication strategy between multiple copies of the coordinator process that behave, from the perspective of an external observer, as if they were a single process.
    
    ***Q:***&nbsp;&nbsp; But isn't that going to be expensive and slow?  
    ***A:***&nbsp;&nbsp; Yes, but if you really need strong consistency, then this is the type of approach you will need to take.

    Alternatively, you might decide that the extra overheads of time, expense and complexity are sufficiently high that you can tolerate a weaker form of consistency.
    

## Consensus: Making Sure Everyone Agrees

Making sure the coordinator process does not fail turns out to be a difficult problem to solve.

At the end of the previous lecture, we mentioned that in the original [Chain Replication paper](https://www.cs.cornell.edu/home/rvr/papers/OSDI04.pdf), one of the first things Renesse and Schneider state is that *"We assume the coordinator doesn't fail!"* &mdash; and they then admit that this is an unrealistic assumption.  They then go on to describe how in their tests, they had a set of coordinator processes that were able to behave as a single process by running a consensus protocol between them.

### When Do You Need Consensus?

Assuming we have a set of processes that must all act in some coordinated fashion, then consensus will be needed in situations such as:

1. ***The Totally Ordered (or Atomic) Broadcast Problem***  
    All processes need to deliver the same set of messages in the same order

1. ***The Group Membership Problem*** or ***Failure Detection Problem***  
    A group of processes must agree some shared state (either their own, or the state of some other group of processes) &mdash; whilst at the same time, any one of these processes could be starting, going slow, crashing or restarting

1. ***The Leader Election Problem***  
    One process plays a distinguished role, and all the others need to know who it is

1. ***The Distributed Mutual Exclusion Problem***  
    All processes take turns in getting mutually exclusive access to some shared resource (E.G. a file)

1. ***The Distributed Transaction Commit Problem***  
    All processes jointly contribute towards a transactional unit of work.  It must then be agreed upon as to whether that unit of work should be committed or aborted

All of these problems share the same common requirement &mdash; they all need to agree on some shared set of information.  Exactly ***what*** needs to be agreed on varies; it could be the order in which messages are delivered, or whether a set of processes are alive or dead, or who the leader is, or who gets access to a shared resource, or whether a transaction should be committed or aborted.  Irrespective of the details, the consensus problem boils down to working out how to agree on information that constitutes  *"common knowledge"*.

What makes this really hard is the fact that faults can occur: namely ***crash faults*** or ***omission faults***.  But even in the somewhat simpler case where we only need to deal with crash faults, this problem is still really hard.

### Now Children, Don't Argue...

Why is it so hard?  Because agreement must be reached:

* In the presence of failure and asynchrony, and
* Without there being any external process to act as arbiter, and
* Without an individual participant knowing what value has been contributed by any of the other participants

So, let's say we have three processes that all need to agree on the value of a single bit - should it be `1` or `0`?  This seems like a simple enough request, but given the above constraints, this is much harder than it appears.

In simple terms, each process contributes its opinion on what this bit should be, and there are only two possible outcomes:

![Consensus Outcomes](./img/L14%20Consensus%201.png)

Irrespective of the individual contributions, all participants must agree on either a `0` or a `1`.

One approach is to use the majority opinion, and this will play a major role when we look into the implementation details, but first let's take a look at the properties that consensus algorithms ***try*** to satisfy.

## Properties a Consensus Algorithm Tries to Implement

In order for consensus to be reached, all algorithms try to implement the following properties:

* ***Termination***  
    Each correct process eventually decides on a value.
    
    Notice that this property makes no attempt to ensure that two correct processes ***agree*** on some common value, only that each correct process will eventually output something.
    
* ***Agreement***  
    All correct processes agree on the same value.

    If these were the only two properties we needed to account for, then we could simply ignore all the input values and output some arbitrary value...  Boom! Consensus!

    But whilst this vacuously conforms to the requirements of consensus, it's not going to be very useful in practice.  So, we need a third consensus property:

* ***Validity*** (or ***Integrity***, or ****Non-Triviality****)  
   The agreed upon value must be one of the proposed values


The problem is that, in reality, it turns out to be impossible for a consensus algorithm to satisfy all three of these properties. This is known as the ***FLP Result*** after a paper published in 1983 by Michael Fischer, Nancy Lynch and Michael Paterson.  In simple terms:

> ***FLP Result***  
> If messages are asynchronous ***and*** crashes can occur, then it is impossible to satisfy all three properties of termination, agreement and validation.

This paper is called the ["Impossibility of Distributed Consensus with One Faulty Process"](./papers/FLP.pdf) and its practical implication is that all consensus algorithms must compromise on at least one of these properties.

The problem is that if we compromise on either the agreement or validity properties, then we will have a system that fulfils the requirements, but possibly only in a vacuous sense and will therefore not be very useful.  Consequently, consensus algorithms must compromise on termination.

With termination compromised, it means that if we get a response, it will always be one of the original input values, but it might take an infinitely long period of time to arrive at an answer...  If that's the case, then we would need to impose some sort of timeout on the consensus algorithm such that if it does not find an answer within a predetermined period of time, we try again &mdash; which also might not terminate, resulting in:

![Non-terminating Consensus](./img/L14%20Consensus%202.png)

Nothing, nada, nix.  But this is a risk we cannot avoid.

## The Paxos Consensus Algorithm

This algorithm was invented by Leslie Lamport in 1990, but it took him 8 years to get it published.  In the original paper, he used the analogy of a parliamentary procedure with laws and voting and ballots; and many people still use this terminology when explaining the algorithm.  However, he later adopted a different set of terminology that was used in his ["Paxos Made Simple"](./papers/Paxos%20Made%20Simple.pdf) paper published in Nov 2001.  This is the terminology we will use here.

Paxos is not a single algorithm; instead it is a family of algorithms.

Here, we are just going to look at the "vanilla" Paxos algorithm.  In this case, a process can act in one (or more) of three roles:

* ***Proposer***  
    Proposes a value
     
* ***Acceptor***  
    Contributes towards choosing from one of the proposed values

* ***Learner***  
    Does not directly participate in negotiating the final value.  Instead, a learner simply learns the value agreed upon by the other participants.  A learner is the only particpant who actually knows the value on which consensus has been reached.

It is entirely possible for Paxos participants to take on two, or even three roles; but to start with, it is simplest to assign one role to one participant.

One of the first things that needs to be known by all the nodes in a Paxos system is how many acceptors are needed to form a majority.

So, if there are three processes performing the role of acceptor, then a majority is two; if there are five, then the majority is three.  This fact must be known ***in advance*** by all participating processes - you can't start running the Paxos algorithm until this knowledge has first been shared.

Another thing is that Paxos nodes need to be able to remember what values they have accepted, so they must have some form of persistent storage.

Another thing to keep in mind here is that we are looking at a protocol for deciding upon a ***single*** value.  If you want to decide upon a sequence of values (which is often needed), then you need to run the protocol again.  There are various optimisations that can be implemented at this point but were not going to talk about those yet.  To start with, we will simply confine ourselves to the process of agreeing upon a single value.

### Paxos Algorithm: A Basic Worked Example

In this example, the participants are:

* A single proposer `P`,
* Three acceptors <code>A<sub>1</sub></code>, <code>A<sub>2</sub></code> and <code>A<sub>3</sub></code>, and
* Two learners <code>L<sub>1</sub></code> and <code>L<sub>2</sub></code>

So, the first thing to note is that a majority of acceptors in this case is two.  This fact was distributed to the participants when they started.

Knowing how many acceptors constitute a majority, the proposer `P` sends out a broadcast `prepare` message to the majority of acceptors.  This message does not contain the value being negotiated, instead it simply contains a proposal number that must obey the following rules:

1. The proposal number must be ***unique***.  
    If the Paxos consensus algorithm is being implemented by multiple proposers, then before the algorithm starts, each of the proposer processes must have already agreed upon how they will maintain proposal number uniqueness (for instance, one proposer might use even numbers and the other, odd).
1. The proposal number must be ***higher*** than any proposal number previously used by that proposer.  
    This means each proposer must keep a record of the last proposal number it used.  Each time the proposer sends out a `prepare` message, the proposal number must be incremented in such a way as to guarantee uniqueness.

![Paxos 1](./img/L14%20Paxos%201.png)

In this case, the proposal number is `5`.  Remember that this is ***not*** the value upon which consensus is being reached.  It is simply an agreed logical clock value that will be used to identify the value upon which consensus is being sought.

The acceptor processes then look at the proposal number and ask the following question:

***Acceptor:*** *"Have I already agreed to ignore messages with this proposal number?"*

If the answer is yes, then this message is simply ignored.  However, if the message is no, then the acceptors reply with a `promise` message containing the value of the proposal number to which they now promise to respond.

This rule contains some subtleties that we're glossing over for now, but for the time being, we'll leave it at this.

![Paxos 2](./img/L14%20Paxos%202.png)

At this point, we've reached something of a milestone because we now have a majority of acceptors who have all agreed that they will respond to an as yet unsent value, that we will identify with this particular proposal number.  In addition to this, there can now never be a majority of acceptors who have promised to pay attention to any proposal number less than `5`. (A minority might still agree on some lower proposal number, that is now of no consequence.)

In other words, we've got enough people paying attention in order to make a decision!  

Now the proposer sends an `accept` message to a majority of acceptors.  For the sake of initial simplicity, we'll choose the same set of acceptors as before.  The `accept` message carries with it both the agreed upon proposal number (`5`) and for the first time, the value upon which consensus is being sought - in this case, let's say it’s simply binary `0`.

At this point in time, we're discussing the simple case in which none of the acceptors dispute the proposed value (more of that in the next lecture)

![Paxos 3](./img/L14%20Paxos%203.png)

As soon as the acceptors receive the `accept` message, they will ask themselves exactly the same question as before:

***Acceptor:*** *"Have I already agreed to ignore messages with this proposal number?"*

If yes, then the message is simply ignored.  If not however, the acceptor does two things:

1. It sends an `accepted` message back to the proposer containing both the proposal number and the accepted value.
1. It broadcasts the `accepted` message to all the learners

So, acceptor <code>A<sub>1</sub></code> responds both to the proposer and broadcasts its acceptance to all the learners.

![Paxos 4](./img/L14%20Paxos%204.png)

Likewise, acceptor <code>A<sub>2</sub></code> does the same thing in parallel.

![Paxos 5](./img/L14%20Paxos%205.png)

### At What Point is Consensus Achieved?

Consensus is reached when the majority of acceptors respond to the proposer with their `accepted` messages:

![Paxos 6](./img/L14%20Paxos%206.png)

The non-intuitive part is that although we identify this as being the point in time when consensus was reached, not everyone knows it yet!

The proposer and learners only learn that consensus has been reached ***after the fact***, because by the time the `accepted` messages arrive from the majority of acceptors, consensus has already occurred.

So, the key point here is that the moment at which consensus is reached is different from the moment when everyone finds out about it.

### What About the Details We've Skipped Over?

There are various subtleties that take place in these algorithms that, at the moment, we have simply glossed over; however, these will be dealt with in the next lecture when we look at the cases in which there is more than one proposer, and what to do in the event of process failure.  We will also look at why the Paxos algorithm is not guaranteed to terminate.

## Summary: Simplified Steps in the Paxos Algorithm

* When an acceptor gets a `prepare(n)` message, it asks *"Have I already agreed to ignore messages with the proposal number `n`?"*
    * If yes, then the message is simply ignored
    * If no, the acceptor now promises to ignore any request with a proposal number less than `n` and it replies to the proposer with a `promise(n)` message.  
        (We've skipped over some details here that we'll need to revisit in the next lecture)
* Once the majority of acceptors have sent their `promise(n)` messages back to the proposer, the proposer responds by sending an `accept(n,val)` message back to the majority of acceptors.  Here, `n` is the agreed upon proposal number and `val` is the proposed value upon which consensus is sought.  
    (Again, we've skipped over some details here that we'll need to revisit in the next lecture)



## Questions

***Q:***&nbsp;&nbsp; How do the acceptors learn that consensus has been reached?  
***A:***&nbsp;&nbsp; Unless the acceptor is also playing the part of a learner, then it actually won't know that consensus has been reached.  In fact, learner processes are the only ones that can know what the agreed upon value is (hence the need for processes to play multiple roles)

***Q:***&nbsp;&nbsp; Why is this algorithm so complicated?  
***A:***&nbsp;&nbsp; The algorithm's complexity is a necessary feature of it needing to be able to withstand process failure.


