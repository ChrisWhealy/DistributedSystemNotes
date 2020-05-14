# Distributed Systems Lecture 18

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 11th, 2020 via [YouTube](https://www.youtube.com/watch?v=xakpenkbOr0)


## Review of the [Amazon Dynamo Paper](./papers/Dynamo.pdf)

Prior to the publication of this paper, the prevailing assumption was that, given the CAP properties of consistency, availability and partition tolerance, it was ***obvious*** that the highest should be consistency.

Amazon said no &mdash; we don't care about consistency; we care about availability.  The main focus of their paper was to describe both how availability can be successfully prioritised over consistency, and to offer an analysis of the consequences of such a reprioritization.

### Availability

We defined availability to be ***"All requests receive a meaningful response"*** and possibly qualify this definition by adding that a response might be given within a certain period of time.

The primary focus of the Dynamo system is to say that if someone wishes to add an item to their shopping cart, then go ahead.  This activity must be prioritised over potential data loss that might occur through some sort of failure.

### Network Partition

A network is said to be partition if machines on that network can no longer communicate with each other.  Such situations are temporary and unintentional.

### Eventual Consistency

This idea is not new, it’s been around since at least the 1990's, but it has experienced a resurgence since the publication of the Dynamo paper to such an extent, that many people think eventual consistency was invented at around this time.  It is common, but incorrect, for people to cite a [2008 article](./papers/Vogels.pdf) by Werner Vogels as the origin of this term.  Instead, cite the older work by Doug Terry explained in articles such as [this one](https://littlemindslargeclouds.wordpress.com/tag/eventual-consistency/).

Eventually consistency can be defined as ***"Replicas will eventually agree if client stop sending updates"***

This consistency model is not however a safety property - it is a liveness property.  This is because a liveness property cannot be violated in a finite execution.

For Dynamo however, it is not immediately clear what safety properties it offers.  We have said that for a system to offer strong consistency, then for nodes that deliver the same set of messages, their states will agree - even if those updates arrive in different orders.  This is clearly a safety property

### Vector Clocks

Dynamo uses a vector clock to give a logical timestamp to events in an attempt to resolve conflicts.  But as we have seen, vector clocks cannot resolve anomalies created by causally independent events.  Dynamo takes two different approaches to conflict resolution:

* For the vast majority pf cases, it sends all the causally independent versions of the data to client, and expects conflict resolution to happen in the application, or
* Much less frequently, it adopts a simple *"last-write-wins"* strategy

### Application-Specific Conflict Resolution

Since the client application has a far better idea of what actions the user is performing, this is the best context within which to handle conflict resolution.  In the case of the user's shopping cart, if it turns out that the server ends up with two or more causally unrelated versions, then all this data is simply passed to the client-side application for resolution.  In the case of shopping carts, the correct approach is for the client simply to create a new shopping cart from a union of the different cart versions.

But why take set union over set intersection?  To sell more products?  Well, maybe, but by taking the union, it guarantees never to miss an item, whereas the intersection guarantees that at least one item will be dropped.  This approach might, occasionally, result in a deleted item popping up again, but in practice, that situation is rare.

Dynamo is designed in such a way that you can elect to implement your own, client-side conflict resolution mechanism, or you can simply delegate this to the server, in which case, the last-write-wins approach is used.

## Write Commutativity

Addition and multiplication are commutative operations because the order of the operands does not change the result.

<code>3 + 4 &equiv; 4 + 3</code>  and <code>3 * 4 &equiv; 4 * 3</code>

It something of an abuse of terminology to describe writes as commutative because this term describes the outcome of a binary operation on the basis of the order in which the operands are applied; however, in spite of our abuse of the terminology, in the case of adding items to a shopping cart, these writes are commutative.  It does not matter the order in which they are applied because a shopping cart is simply a set in which there is no causal relationship between the members.  The presence of a book in your shopping cart is unrelated to the presence of a pair of jeans.

This fact alone is enough to gives us strong convergence; however, the paper also talks about vector clocks.  Why are vector clocks needed when we've just established that there's no causal relationship between the items in a shopping cart?

Why are vectors clocks needed?

They're needed because addition is not the only operation you can perform on the items in a shopping cart.  Causal anomalies can occur if deletions are not processed correctly.  So, if, by means of a vector clock, we can demonstrate that version `A` of a shopping cart is the causal antecedent of version `B`, then version `A` can be safely thrown away.  However, if no causal relationship can be established between versions `C` and `D`, then divergence has occurred and both versions of the shopping cart must be sent to the client for resolution.

## Resolving Replica Disagreement

It’s one thing for a disagreement to exist between replicas, but ***recognising*** that such a disagreement exists is also a challenge.

By drawing a Lamport Diagram, we are taking a God's-eye view of the situation and from this perspective, disagreements are obvious.

![Total-Order Anomaly](./img/L12%20TO%20Anomaly.png)

### Anti-Entropy Protocol

However, without some additional strategy in place, neither replica can tell that a disagreement exists.  Dynamo handles conflict is replica state using an anti-entropy protocol based on Merkle Trees.

What this means in practice is that once per second, every node picks another node at random, and those two nodes compare their state.  If they discover difference, then they start exchanging more detailed state information until the conflicting key/value pair is located.

### Gossip Protocol

A gossip protocol is different from, but related to, an anti-entropy protocol.

In Dynamo, data is partitioned across a set of nodes organised in a ring.  At any time, a node may fail, or a new node might be added to that ring, so the gossip protocol is used to resolve membership conflicts in the ring.

Using the gossip protocol, each node shares its own membership information with the others, and additionally, each node needs to know who the other current ring members are and what role they play.

So, we have two related, but distinct forms of conflict resolution protocol at work here:

* The anti-entropy protocol resolves conflicts in the shared state of each node's key/value store  
    I.E. Questions related to *"What do you think the value of `x` is?"*

* The gossip protocol resolves conflicts in ring membership  
    I.E. Questions related to *"Who do you think is up and running?"*


One thing the Dynamo paper states is that Amazon used to maintain a global view of the state of each node in a Dynamo system; however, after a while, they discovered that this global view was not needed (probably because it created some sort of bottle-neck) and that a local view was sufficient.

Both of these protocols are examples of *eventually consistent* protocols that implement strong convergence in application state.

> ***ASIDE***
> 
> In the context of the Dynamo Paper, the terms ***anti-entropy*** and ***gossip*** are used to describe two different mechanisms for resolving distinct, but related types of conflict.
> 
> However, in other papers, you may find that these terms are used synonymously; with "anti-entropy" being used as a fancy term for "gossip"

### Are These Two Types of State Conflict So Different?

Conceptually, resolving a conflict in the shared state of a key/value store, and resolving a conflict of group membership are quite similar.  The key difference lies on how much data needs to be exchanged firstly, to discover a difference, and secondly to resolve it.

In terms of discovering differences in group membership, only a small amount of data needs to be exchanged before a difference is discovered and then resolved.  However, in the case that a key/value store must be shared between multiple nodes, a potentially huge amount of data might need to be exchanged before a difference is discovered.

### Using Merkle Trees to Discover State Differences

In order to exchange state data efficiently, Dynamo minimises the amount of network traffic by using Merkle (or Hash) Trees.  A Merkle Tree is a binary tree in which the value of each parent node is the hash of the values of its children.  Thus, a difference in the state of the entire key/value store can be discovered simply by two nodes comparing the hash value of the root nodes of their Merkle Trees.  If these values differ, then somewhere further down the tree, a difference must exist.  Only then do the nodes start exchanging lower level hash values until the different key/value pair is discovered.

This solves the "*cost of data transfer*" problem.

![Merkle Tree 1](./img/L18%20Merkle%20Tree%201.png)

The above diagram represents the Merkle Tree that would be created for a keystore containing four values.  In order two nodes to discover is their keystores were identical, all they would need to do is compare their root node values of `H7`.  If these are identical, then everything else beneath that point in the tree must also be identical.

If two data stores do differ in only one item, then the difference can be discovered by traversing the Merkle Tree until the differing leaf node value is discovered.

![Merkle Conflict 1](./img/L18%20Merkle%20Conflict%201.png)

Here, we can see that the value of `C` differs between replicas 1 and 2.  Consequently, the value of all the parent nodes above `C` will differ.  So, when replicas 1 and 2 compare root node has values, they will discover that these values differ (the difference here is illustrated using colour rather than any numerical value):

![Merkle Conflict 2](./img/L18%20Merkle%20Conflict%202.png)

The root nodes are compared and found to be different, so the child node values are compared

![Merkle Conflict 3](./img/L18%20Merkle%20Conflict%203.png)

The `H6` node in each tree is the same, so the difference cannot lie beneath this node.  So, this side of the branch can be ignored.  However, the `H5` node value differ, so the difference must lie somewhere beneath this node.

![Merkle Conflict 4](./img/L18%20Merkle%20Conflict%204.png)

The children of `H5` are compared, and the difference is found to be with node `H2`.

![Merkle Conflict 4](./img/L18%20Merkle%20Conflict%204.png)

The child of `H2` turns out to be a leaf node.

![Merkle Conflict 5](./img/L18%20Merkle%20Conflict%205.png)

So, this is where the differing key/value pair is located.  Now we have established that replica one thinks `B=2`, but replica 2 thinks `B=9`

So, the Merkle Tree comparison strategy *"binary chops"* its way down the tree to locate the differing value.

We should also note that the values stored in the keystore are not necessarily simple integers as shown here; they are usually entire data objects. So, this strategy identifies the exact differing value, and then only that value needs to be sent over the network.

If more multiple key values differ, then these can also be discovered using the same strategy.  In the worst case, you would need to compare all the values in the keystore; but this is a highly unlikely situation.  In reality, the differences between large keystores are small, thus making this a various efficient difference detection strategy - especially when these differences need to be discovered over a network.

Merkle Trees are used in a wide variety of situations, not only for comparing keystore values.  They are often used in the case of authentication.

In the case of Dynamo however, they explicitly state that this strategy is employed within a trusted, non-threatening environment.  So, under these conditions, the primary consideration here is to save on network bandwidth.  Attacks from hostile third parties are explicitly excluded from this discussion; therefore, there is no need for Dynamo to have any concept of user or request authentication.  However, Merkle Trees are useful in both situations.

### Resolving State Differences

This is a very different question from detecting that replicas disagree with each other.  The use of Merkle Trees is confined simply to making difference detection as efficient as possible.  Once a difference has been discovered however, we must now engage in a very different type of conflict resolution, and this is application specific.

Once the Merkle Tree has been used to identify differing key/value pairs, the difference can be resolved at the keystore level simply by storing both versions of the object, so in the example we used above, the resolved state of the keystore would be:

![Merkle Tree 2](./img/L18%20Merkle%20Tree%202.png)

Now, when a client wants to read the value of `B`, it would receive the set of values held in the keystore, and given that the application understands the meaning data, it is typically responsible for  deciding which version of the object is the right one.  It might do this on the basis of something like a timestamp or a vector clock carried in the object's metadata.


One question that often comes up here is *"What if the two replicas hold completely different key/value pairs?"*.

In the case of the Dynamo paper, this problem does not really become an issue because when a ring of nodes is started, each node is informed about the key range that it will be responsible for.  Therefore, significant structural differences between replica keystores are minimal.

However, more generally, the Merkle Tree does not need to be specifically a binary tree.  It could be a Merkle Tree that uses addition prefixes (known as a *Merklix Tree*).  Alternatively, the Merkle Tree could use a branching factor higher than two.


## Quorum Consistency

In the remaining time in this lecture, we will only have time to touch this topic briefly.

In Primary Backup replication and Chain Replication, the clients are restricted to talking only to specific nodes:

* In Primary Backup Replication, the clients always talk to the primary node for both reads and writes
* In Chain Replication, the clients talk to the head node for reads and the tail node for writes

However, in Dynamo the clients can talk to ***any*** node.  In fact, the clients typically update their node table every 10 seconds, and based on this information, the client calculates which node to talk to, based on who gave the fastest response to their previous operation.

The question now becomes this: *"How many replicas should I talk to before getting an answer I can consider the answer to be reliable?"*

In a quorum consistency environment, there are three specific, configurable values that control how this question should be answered.

* `N` - The number of replicas (typically 3)
* `W` - **The Read Quorum**  
    The number of replicas that must respond to a write operation in order to consider that operation a success (typically 2)
* `R` - **The Write Quorum**  
    The number of replicas that must respond to a read operation in order to consider result reliable (typically 2)


If we suppose that `W` is set to 3 and `R` to 1, then we have implemented a system that prioritises read performance.  This is because firstly, we require all three replicas to acknowledge the success of a write (thus providing strong consistency), and secondly, knowing that all the replica ***must*** contain the same data, we can be sure that a read from any replica will be authoritative.

This is a popular Dynamo configuration setting, often known as Read One, Write All (or ROWA) and this does provide reliable writes and fast reads, but is it fault tolerant?

No, it’s not because if one of the nodes crashes, or a network partition suddenly separates the client from one of the replicas, then it can no longer perform any writes.

So, whilst this gives certain advantages, it does so by carrying the risk of reduced fault tolerance.
