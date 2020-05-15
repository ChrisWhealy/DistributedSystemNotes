# Distributed Systems Lecture 19

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 13th, 2020 via [Twitch](https://www.twitch.tv/videos/619837000)

## Continuation of Quorum Consistency

We finished the last lecture with this overview.

In a quorum consistency environment, there are three specific, configurable values that control how this question should be answered.

* `N` - The number of replicas (typically 3)
* `W` - **The Write Quorum**  
    The number of replicas that must respond to a write operation in order to consider that operation a success (typically 2)
* `R` - **The Read Quorum**  
    The number of replicas that must respond to a read operation in order to consider result reliable (typically 2)

Depending on what balance our system needs to have between availability and consistency, we need to tune the `NWR` values accordingly.

For instance, if we want strong consistency, then we could use `N=3, W=3, R=1` meaning firstly that a write operation is only considered successful if it has been acknowledged by all three nodes.  A direct consequence of setting `W=N` is that we now know for certain that all the nodes will agree with each other; therefore, we can set `R=1` knowing that it doesn’t matter which node we read from, because they all agree.

Whilst this is a popular configuration setting, the trouble is that if any of the write nodes crash, or a network partition suddenly appears, you're unable to perform any further writes.

Instead of `N=3, W=3, R=1`, the Dynamo paper recommends `N=3, W=2, R=2`

This now creates the potential situation that a read conflict could occur.

![Dynamo Read Conflict 1](./img/L19%20Dynamo%20Read%20Conflict%201.png)

A client changes the new value of `x` from `4` to `5` by writing to replicas `R1` and `R2`.

If the client then immediately reads the value of `x`, it is possible that before all the nodes have synchronized their values, this read operation might be handled by replicas `R2` and `R3`.

![Dynamo Read Conflict 2](./img/L19%20Dynamo%20Read%20Conflict%202.png)

This will result in the client receiving different values of `x`.

At this point, it is important to understand that there is a discrepancy between different system implementations about what the read quorum value `R` means in practice.  Does is refer to:

* The number of nodes that must respond to a read request, or
* The number of nodes that must respond to a read request *with the same value*?

In the context of the Dynamo paper where Amazon prioritise availability over consistency, it seems clear that the read quorum value `R` is used to refer to the number of nodes that must respond to a read request.  If they return conflicting values, then so be it.

In general, though, if we ensure that `R + W > N`, then we can be certain that every write quorum intersects with every read quorum, thus ensuring that we avoid what's known as a *"stale read"* where all of the responses are out of date.  The `R + W > N` approach ensures that a read will always return at least one correct response.

For instance, if we set the write quorum to `2` and the read quorum to `1`, then it’s possible that we could be writing to replicas `R1` and `R2` and reading from replica `R3`.  Thus, until the nodes synchronise their state, all our read operation could return stale values.

A further caveat is to understand that simply satisfying the inequality `R + W > N` only helps prevent stale reads, it does not ensure fault tolerance because as we saw in the first example, setting `N=3, R=1, W=3` guarantees strong consistency at the expense of fault tolerance.

The Dynamo paper quotes the configuration settings as `N=3, R=2, W=2` because this ensures that someone will respond to a read operation with the correct value.  In Amazon's situation, it is typically the client application that must then resolve the conflict.

***Q:***&nbsp;&nbsp; What's wrong with the "Read One, Write All" approach?  
***A:***&nbsp;&nbsp; Several reasons.  Firstly, to ensure strong consistency is slow.  Since we have to wait for all the nodes to acknowledge a write operation, the write response time cannot be any faster than the slowest node.  Also, this configuration is not fault tolerant.  If a node crashes or a network partition suddenly appears, then immediately, we have lost the ability to performs writes.

## Sharding or Data Partitioning

Here's one of those annoying situations in distributed systems where the same term is used to mean two, totally different things.  The word *"partitioning"* here is used to mean the way you split up your data between different nodes (which is a good thing), as opposed a sudden loss of communication caused by a network partition (which is a bad thing).

In the systems we've spoken of so far, all the machines store all the data.  So, each machine has a full copy of your entire keystore.

![No Sharding](./img/L19%20No%20Sharding.png)

But this approach has several drawbacks:

* ***Scalability***  
    If you're running primary backup replication, then under heavy load conditions, all the work falls on the primary which soon becomes a bottleneck
* ***Quantity***  
    Maybe you have more data than will fit on a single machine
* ***Synchronisation***  
    Each copy must synchronise with all the others every time there is a write

But how about an approach like this?

![Sharding 1](./img/L19%20Sharding%201.png)

So now, we don't have the problem of stale reads because each machine only handles a known subset of the data.  The trade-off here is that in this example, we have now lost fault tolerance because there is no replication.  But we could reinstate replication quite easily.

![Sharding 2](./img/L19%20Sharding%202.png)

This technique is known as ***Sharding*** or ***Data Partitioning***.

Each fragment of data stored on a replicated node is known as a ***shard*** or a ***replica group***.

![Sharding 3](./img/L19%20Sharding%203.png)

There are multiple reasons for why you might want to adopt this approach.  These include:

* If the range of client requests is more or less evenly distributed across the full range of keys in your keystore, then this will improve throughput
* Or, you might simply have too much data to fit on one machine, in which case, sharding is the only viable option

The design shown above is somewhat simplistic in that we have a single key/value pair stored per shard.  In reality however, shards will typically store broad ranges of key/value pairs.  Even with this taken into account, the arrangement shown above still does not represent the way Dynamo implements sharding.

Here's another arrangement that combines these various approaches into one:

![Sharding 4](./img/L19%20Sharding%204.png)

Each node now holds an overlapping range of key/value pairs.  `X` can be found on `M1` and `M2`, `Y` can be found on `M1` and `M3`, and `Z` can be found on `M2` and `M3`.  So, it looks like we have the best of both worlds here: we have replication across nodes and sharding.

In fact, what we have done is implement a distributed version of Primary Backup Replication.  In the previous examples of PB Replication, we had a single node acting as the primary and all the data lived on all the nodes.  Consequently, the primary had to handle all the reads and writes and could become the bottleneck under high load situations.

By sharding the data, we have said that one node can act as the primary - not for all the data, but just a subset of that data.  So now the role of primary and backup has been distributed across multiple nodes.

![Sharding 5](./img/L19%20Sharding%205.png)

So, each node plays the role of primary for some values and backup for others.

| Key | Primary | Backup
|---|---|---
| `x` | `M1` | `M2`
| `y` | `M3` | `M1`
| `z` | `M2` | `M3`

Given this division of labour, when the client wants the value of `x`, it will talk to the node acting as the primary for `x` - which in the case, happens to be `M1`.  Similarly, a client wanting the value of `z` will talk to `M2`.

Not only have we split up the workload of serving requests, but each machine now only needs to store two thirds of the entire dataset.

The only remaining detail to consider is that each primary must replicate all write requests to their respective backup node(s).

![Sharding 6](./img/L19%20Sharding%206.png)

No matter how the sharding is implemented, we have two forms of replication at work here:

![Key Replication](./img/L19%20Key%20Replication.png)

![Dataset Replication](./img/L19%20Dataset%20Replication.png)


The choice for how you replicate data and how it is sharded are somewhat orthogonal to each other.  In the rest of this discussion, we will focus on sharding techniques and assume that replication will be implemented somehow.

So, let's now just focus on partition strategy

## Partition Strategies

How do you decide which key/value pairs go where?  To answer this question, we must first establish what goals our partitioning strategy needs to meet.

We really need to achieve two goals here:

* ***Goal 1***&nbsp;&nbsp; Avoid read or write hotspots
* ***Goal 2***&nbsp;&nbsp; Make the data easy for clients to find quickly

| | Goal 1 | Goal 2
|---|---|---
| Random distribution of data across all the nodes | ![Tick](./img/tick.png) | ![Sad face](./img/emoji_sad.png)<br>The clients now have no idea where the data lives...
| Put all the data on one node | ![Sad face](./img/emoji_sad.png)<br>Under high load conditions, this creates a performance bottleneck | ![Tick](./img/tick.png)


So, neither of these are good ideas.

### Partitioning Data by Key Range

If we know that the key values in our dataset fall into some range, then we can distribute that data by allocating each node a key range.

So, if you're handling data that is sorted alphabetically, then you could distribute the data across three machines such that keys starting with a particular letter will be handled by a known machine.

| Alphabetic Range | Machine
|---|---
| `a-h` | `M1`
| `i-r` | `M2`
| `s-z` | `M3`

But there's a big problem here: unless the data is uniformly distributed across the key range, then you will still have hotspots.

How do we take data that has a non-uniform distribution within its key range, and map it some other domain such that the mapping process yields the uniform distribution we're looking for?

Hashing!

### Partitioning Data by Hash Function

A good hashing function such as MD5 will distribute its input space evenly across its output space.

The MD5 algorithm returns a number in the range `0` to <code>2<sup>128</sup> - 1</code>.  For instance

![MD5 Output Space](./img/L19%20MD5%20Output%20Space.png)

What we need to do now is make sure that each node has an evenly sized chunk of the has function's output space.  However, and easier way to calculate this is to say that since we wish to distribute the hash values across 3 machines, then we can simply calculate `modulus 3` of the has function value to give a server number of `0`, `1` or `2`.

So, we can extend this diagram 

![MD5 Has to Node Number](./img/L19%20MD5%20To%20Node.png)

This is the *"Hash mod `N`"* approach and as good as it is, it has a significant drawback.

The problem is that if `N` changes, then this completely alters the target machine not only for new key values, but all the existing key values.

For instance, if we added a new machine `M4`, then this could result in having to move our `"aardvark"` from `M1` to `M0` &mdash; and he's just settled into his new home and now you want to move him again...

It is certainly true that when a new machine is added, some of the existing data will need to be relocated, but it does not makes sense to move existing data from one old node to another old node.  In an ideal situation, the addition of a new node should trigger the minimal amount of data transfer, and certainly not cause data to be moved between existing nodes.

So, what is the minimum amount of necessary data transfer after the addition of a new node?

Let's say we have 6 keys distributed over 3 nodes.  This then averages out to 2 keys per node.

![Node Addition 1](./img/L19%20Node%20Addition%201.png)

Now, due say to a spike in request volume, we need to add a new node `M4`.  If we still wish to maintain an even distribution of keys per node, then we need to transfer some of the data to this new node.  But how many keys should we move?

In general, if we have `K` keys distributed across `N` nodes, then on average, each node will store `K/N` keys.  So previously `K=3` and `N=6`, so no prizes for figuring out that each node should hold 2 keys.  If `K` now increases to 4, the `K/N` is `1.5`.  Since we can't move half a key, we'll round this value down.  So, in general we can say that each node will hold `floor(K/N)` keys.

![Node Addition 2](./img/L19%20Node%20Addition%202.png)

The addition of the new node has caused a single key to be transferred to the new node (don't move the aardvark!).

Notice that data is only transferred from an old node to the new node; no data is transferred between the old nodes.

### Consistent Hashing

This is yet another case where a word that we've previously used to mean one thing ("*consistent*") is now being used to mean something different - but the use of the word "consistent" here has its origins in networking, not distributed systems.

The first change we need to make is to arrange our nodes in a ring.  The position of a node around the ring corresponds to some range of values from our hashing function's output space.

The MD5 function has a vast output space ranging from `0` all the way up to <code>2<sup>128</sup> - 1</code>.  This is such a huge number that we cannot represent it graphically in any meaningful way, so for the purposes of our diagram, let's compress the hash function's output space down to the range `0` to `63`.

![Dynamo Node Ring 1](./img/L19%20Ring%201.png)

So, in this scenario, our reduced hash function has positioned our four nodes roughly evenly around the ring at the following locations

* `M1` is at `8`
* `M2` is at `20`
* `M3` is at `32`
* `M4` is at `47`

Let's now say we want to add the new key `"apple"` and the hash function yields `14`.

![Dynamo Node Ring 2](./img/L19%20Ring%202.png)

There is no node sitting exactly at location `14` on the ring, so we scan clockwise around the ring looking for the next available node, which in this case, turns out to be `M2`.

The rule here is that keys belong to their clockwise successor on the ring.

Now let's say we want to visit our old friend the Aardvark.  In our particular scheme, `"aardvark"` hashes to `62`, so we repeat the same process as before:

![Dynamo Node Ring 3](./img/L19%20Ring%203.png)

There is no node at `62`, so we continue clockwise around the ring, passing go, collecting $200 and encounter `M1`.  So, `"aardvark"` will be stored in node `M1`.

Dynamo also uses this scheme to decide where a value should be replicated.

If we assume that the ring's replication factor is 3, then this means every key must be stored on a total of 3 nodes.  How do we decide which three nodes?

The approach here is the following:

* Every key has a *"home"* node.  This is the node the key is stored on using the above methodology.
* Once the key is stored on the home node, it is then passed on to the next node in the ring and stored there too.
* The key continues to be passed around the ring until it has been stored in the correct number of replicas.

So, in the example where we stored `"apple"` on `M2`, the replication factor of 3 requires that this key is also be replicated nodes on `M3` and `M4`

![Dynamo Node Ring 4](./img/L19%20Ring%204.png)

The Dynamo Paper refers to this list of nodes as the *"preference list"*, and this is usually longer than replication factor indicates because nodes could be down of for some other reason, is down.  So, in Dynamo, you keep working your way down the preference list using the first available nodes required to satisfy the ring's replication factor

### Adding a New Node

We started this discussion in order to understand how we could move as little data as possible when a new node is added to the ring.

As it currently stands, whenever a new key is added to our ring, it will be land on the home node calculated from the following hash function ranges:

| Hash Function Range | Home Node |
|---|---|
| `09` to `20` | `M2`
| `21` to `32` | `M3`
| `33` to `47` | `M4`
| `48` to `08` | `M1`

Ok, but now let's add a new node `M5` at position `60`.  How does this change things?  How much data will we need to move?

| Hash Function Range | Home Node |
|---|---|
| `09` to `20` | `M2`
| `21` to `32` | `M3`
| `33` to `47` | `M4`
| `48` to `60` | `M5` &lt;&mdash; New node
| `61` to `08` | `M1`

So, effectively, we have taken `M1`'s hash function range and split it in half. The range of keys with hash values between `48` and `08` would previously all have landed on `M1`, but now `M5` has arrived at location `60` and taken over the hash function values in the range `48` to `60`.  Therefore, the only keys that need to be moved are the keys currently stored in `M1` that have a hash function values in the range `48` to `60`.

Nothing else needs to change.

![Node Addition 3](./img/L19%20Node%20Addition%203.png)

None of the other nodes are affected.  In fact, these other nodes do not even need to know that a new node at some other, distant part of the ring has been added.

### What Happens if a Node Crashes?

In our diagram, let's look at the consequences of node `M2` crashing.  What do we already know?

* `M2` is responsible for keys having hash values in the range `09` to `20`
* `M3` is responsible for keys having hash values in the range `21` to `32`
* `M3` is the successor of `M2` and our ring uses a replication factor of 3; therefore, all the keys that use `M2` as their home node have ***already*** been replicated in both `M3` and `M4`

![Node Crash 1](./img/L19%20Node%20Crash%201.png)

All that happens is that `M3` simply extends its hash value range to include `M2`'s range.  `M3` hash value range is now from `09` to `32`.

All of `M2`'s keys backed up in `M3` now become the primary copies of that data, and any new key values in the range `09` to `32` are written directly to `M3`.

At this point in time, the administrator will probably want to bring up a new node to replace `M2`, but until they do, `M3 ` takes up the slack.

### Can Consistent Hashing Go Wrong?

Yes.  If the nodes are not evenly spread around the ring, a node could become overloaded and potentially crash.  If your input values fall into a narrow range, then there will not be a particularly good distribution of nodes around the ring.

One trick that Amazon use in Dynamo is the idea of virtual nodes.  This is where, instead of mapping a node to a single location on the ring, it is mapped to a set of locations.  So, one actual node instance, could be mapped to 10, or 20 or 50 locations around the ring.  Using this trick, you are much more likely to achieve an even distribution around the ring.  

In addition to improving the distribution of nodes around the ring, you could have a different number of physical nodes per virtual node to account for differing storage capacity on the physical hardware where that node is running.

For instance, if the hardware on which one node is running has a 1Tb hard drive, then you might allocate 20 virtual nodes to this one physical node.  However, another node in the same ring is running on a machine with a 2Tb hard drive, you might allocate 40 virtual nodes knowing that this machine can handle a bigger hash value range due to its increased storage capacity.

Unfortunately, the use of virtual nodes has a couple of downsides.

Firstly, if a physical node goes down that has lots of virtual nodes assigned to it, then this has a large impact on the rest of the ring, because it appears that suddenly, lots of "nodes" have gone down.  Now lots of other nodes will need to take over has value ranges that are no longer being looked after by these virtual nodes.

The Dynamo paper, however, declares this to be a feature because if a physical node goes down that is running multiple virtual nodes, then those node's workload is spread out around the ring, rather than being taken on a single successor.  Although this does tend to make sense, it’s somewhat harder to reason about.

Secondly, replication of virtual nodes is more complicated because you don't want to replicate data to two different virtual nodes if those nodes are running on the same physical machine, because then although you've replicated your data, you haven't achieved fault tolerance because the data still lives on the same hardware.



## Finally, A Word from Our Sponsor

Remember, be nice to Aardvarks...

![Baby Aardvark](./img/aardvark.jpeg)


