# Distributed Systems Lecture 22

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 29<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=qmFoSeSvarA)

| Previous | Next
|---|---
| [Lecture 21](./Lecture%2021.md) | [Lecture 23](./Lecture%2023.md) 


## Keeping Replicas Consistent

Consider the following scenario in which we need strong consistency, but we have:

* A datastore held across multiple replicas
* These replicas have no leader nor coordinating process (I.E. We're not using Primary Backup or Chain Replication)
* Any replica can receive an update

The challenge then concerns how to keep these replicas consistent with each other.

What approach should we adopt?

Well, we could use a consensus protocol to decide which updates should processed in which order.

![Replica Consensus 1](./img/L22%20Replica%20Consensus%201.png)

If replica `R1` receives update `A` and replica `R2` receives update `B`, then an agreement must be reached concerning the order in which these updates should be delivered.

So, even though the messages will arrive in some unpredictable order, a consensus protocol will be used to decide upon the delivery order.  In this case, both replicas operate a system of delivery slots and the consensus protocol determines that event `B` should occupy delivery slot `1` and event `A` should occupy delivery slot `2`.

***Q:***&nbsp;&nbsp; But how many messages need to be sent in order to arrive at this agreement?  
***A:***&nbsp;&nbsp; Lots!

Here's an example showing the messages that need to be exchanged for replicas <code>R<sub>1</sub></code> and <code>R<sub>2</sub></code> simply to agree on a total order for delivering events `A` and `B` shown above.

![Replica Consensus 2](./img/L22%20Replica%20Consensus%202.png)

1. Replica <code>R<sub>2</sub></code> sends out a `prepare(6)` message to a majority of acceptors, who each respond with the corresponding `promise` messages.  
    4 messages
1. Just a little time after <code>R<sub>2</sub></code>'s message exchange has taken place, replica <code>R<sub>1</sub></code> sends out its `prepare(5)` messages to a majority of acceptors.  Acceptor <code>A<sub>1</sub></code> happily accepts this proposal number, but acceptor <code>A<sub>2</sub></code> has already promised to ignore messages with a proposal numbers less than `6`, so this `prepare` message is ignored and replica <code>R<sub>1</sub></code> left hanging.  
    3 messages (all of which turn out to be redundant)
1. Replica <code>R<sub>2</sub></code> sends out its `accept(6,(slot_1,B))` messages to the acceptors who each respond with `accepted(6,(slot_1,B))`.  So, event `B` now occupies delivery slot 1.  
    4 messages
1. Replica <code>R<sub>1</sub></code> still needs to get agreement on a total order for event `A`, so it tries the prepare/promise phase again, but now with proposal number `7`.  This time, the proposal number is accepted.  
    4 messages
1. Replica <code>R<sub>1</sub></code> then enters the accept/accepted phase and achieves consensus on event `A` occupying delivery slot 2.  
    4 messages

So, even in this reasonably happy example where we are not sending messages to all the acceptors (only a majority), we've had to send 19 messages, 3 of which turned out to be redundant &mdash; and we haven't even accounted for sending messages out to the learners!

The point here is that consensus algorithms are really expensive; therefore, we should only implement them in situations where it's ***extremely*** important that everybody agrees on a total order.

So far however, we have not even discussed what events `A` and `B` represent.  Consensus algorithms do not care about a message's payload; they simply see an opaque (I.E. meaningless) block of data to which some metadata has been attached.  Causal Broadcast for instance, looks simply at the message's recipient and the vector clock value, and from these values, determines the delivery order.

> ***My Aside***
> 
> A useful analogy here is to think of the people working in a mail sorting room.  These people are concerned with the fact that all the letters and packages have been addressed correctly, and that the correct postage has been paid for a letter or package of that weight and dimensions.
> 
> It is quite irrelevant for these people to concern themselves with the contents of the letters and packages.

However, from the perspective of the human developer, we can see firstly that we're having to go to a lot of trouble simply to agree on the order in which a set of events are delivered, and secondly, the algorithm that determines the delivery order is completely agnostic to the real-life functionality that needs to be performed as a result of processing those events.

Rather than having a message-agnostic consensus algorithm then, wouldn't it be smarter to make intelligent decisions about delivery order based on our knowledge of the functionality being implemented?

### Safety Properties: Do We Really Need Strong Consistency?

Let's go back to the shopping cart scenario described in [lecture 17](./Lecture%2017.md).

![Amazon Shopping Cart 1](./img/L17%20Amazon%20Cart%201.png)

If replicas `R1` and `R2` simply represent shopping carts, then we certainly don't want to go to all the trouble of running a consensus algorithm.  But we have arrived at this conclusion based on our knowledge of the business process we are implementing.  In this case, we really don't care whether a book is added to the shopping cart before or after the pair of jeans.  From the perspective of the business logic, the order is neither here nor there.

Of course, there will be some situations in which message delivery order is critical to the logic of your business process, but what we propose here is that strong consistency is needed only in a minority of cases; the greater majority of business scenarios will function quite happily with ***strong convergence***.

Just as a reminder:

***Strong Consistency***  
If replica `R1` delivers messages in the order `M1`, `M2` and `M3`, then all replicas receiving the same set of messages must deliver them in the same order.  Only then can it be known that the replicas have equivalent state.

***Strong Convergence***  
All replicas delivering the same set of messages eventually have the equivalent state.

Strong convergence might still be tricky to implement, but it will be easier than strong consistency, because with strong convergence, we know that state equivalence can be achieved simply by delivering the same set of updates.  Strong consistency however requires us to deliver the same set of updates in ***precisely the same order***.

The bottom line here is that you should only implement strong consistency when you have no other choice.

## How Do We Generalise the Requirement for Strong Convergence?

To do this, we will need to look back at the definition of a partially ordered set that we covered in lectures [3](./Lecture%203.md) and [4](./Lecture%204.md).

As a reminder, a partial order allows you to compare the members of set `S` using a binary relation such as `≤` (less than or equals).  However, the word *"partial"* in the name tells us that not every pair of elements in the set is comparable.

This relation is governed by three axioms:

| Property | English Description | Mathematical Description |
|---|---|---|
| Reflexivity   | For all `a` in `S`,<br>`a` is always `≤` to itself | `∀ a ∈ S: a ≤ a`
| Anti-symmetry | For all `a` and `b` in `S`,<br>if `a ≤ b` and `b ≤ a`, then `a = b` | `∀ a, b ∈ S: a ≤ b, b ≤ a => a = b`
| Transitivity  | For all `a`, `b` and `c` in `S`,<br>if `a ≤ b` and `b ≤ c`, then `a ≤ c` | `∀ a, b, c ∈ S: a ≤ b, b ≤ c => a ≤ c`

The standard example of a partially ordered set is set inclusion &mdash; that is the set of all possible subsets of a given set.

For example, if we have a set containing:

| | | 
|---|---|
| ![Book](./img/emoji_book.png) | A book
| ![jeans](./img/emoji_jeans.png) | A pair of jeans
| ![Torch](./img/emoji_torch.png) | A torch

Then the inclusion set (the set of subsets) will contain the following eight members:

![Set of Subsets](./img/L22%20Set%20of%20Subsets.png)

Our relation here is the *"less than or equals"* operator `≤`.  Using this operator, certain members of our set of subsets can be compared as follows:

![Comparable Subsets 1](./img/L22%20Comparable%20Subsets.png)

However, other members are not comparable; for instance:

![Noncomparable Subsets](./img/L22%20Noncomparable%20Subsets.png)

### Upper Bounds

If we select some elements from our set of subsets, say the singleton set containing the jeans `{👖}` and singleton set containing the torch `{🔦}`, we could ask the following question:

> Which elements of `S` are at least as big as `{👖}` and  `{🔦}`?

The answer here is:

> Any set that contains at least the union of `{👖}` and `{🔦}`.

So, this would be the sets `{👖,🔦}` and `{📓,👖,🔦}`.  These two sets are known as the ***upper bounds*** of `{👖}` and `{🔦}`.

In more formal language, the upper bound is:

> Given a partially ordered set<sup id="a1">[1](#f1)</sup> (`S`, `≤`) an upper bound of `a,b ∈ S` is an element `u ∈ S` such that `a ≤ u` and `b ≤ u`.

Notice that we talk of ***an*** upper bound.  This means it is possible that for the members `a` and `b` there could well be multiple examples of some set `u` that all satisfy the upper bound requirements.

### Which Upper Bounds are Going to Be the Most Interesting?

The upper bound set that contains all the members of the original set is not very interesting because this will always be a common upper bound for all its subsets, so we can ignore this one.

Generally speaking, the upper bounds that are the most interesting are the smallest ones.  But how do we define the smallest upper bound?  The formal definition is:

>  If `a`, `b`, `u` and `v` are all members of the inclusion set `S`, then `u` is the least upper bound<sup id="a2">[2](#f2)</sup> of `a,b ∈ S` if `u ≤ v` for each `v`

### Join-semilattice

***Q:***&nbsp;&nbsp; If `S` is the eight-member inclusion set of `{📓,👖,🔦}`, then is it true that every 2 elements of `S` will have a least upper bound (lub)?  
***A:***&nbsp;&nbsp; Yes!

Any set for which this property is true is given the fancy name of a ***Join-semilattice***.

> A partially ordered set (poset) in which every 2 elements have a least upper bound (lub) is called a join-semilattice.


So, it follows therefore that if some partially ordered sets are join-semilattices, then there should also be some partially ordered sets that are not.

It's quite hard to think of a set within which every 2 elements ***do not*** have a least upper bound (I.E. think of a set that is not a join-semilattice), but a good example is a Boolean register.  This is a tri-state variable that can be either `empty`, `true` or `false`

So, the poset is simply `{empty, true, false}`.  This is a very simple set containing only three members that can be arranged as follows:

![Boolean Ordering](./img/L22%20Boolean%20Ordering.png)

These values can also be ordered using the `≤` operator:

```
empty ≤ empty
 true ≤ true
false ≤ false

empty ≤ true
empty ≤ false
```

So, is this a true partially-ordered set?  To answer this question, we must check that all the axioms are satisfied.

***Reflexivity***  
Since `empty ≤ empty`, `true ≤ true` and `false ≤ false`, then this set satisfies the requirements of reflexivity.

***Anti-symmetry***  
Anti-symmetry requires that if `a ≤ b` and `b ≤ a`, then `a = b`.  However, since this set contains only three members arranged in a two-layer lattice, we have already implicitly satisfied anti-symmetry by satisfying the requirements of reflexivity.

***Transitivity***  
Transitivity requires that if `a ≤ b` and `b ≤ c` then `a ≤ c`.  However, no members of the set can be compared this way, so this set obeys transitivity only in a vacuous sense.

So, this Boolean Register qualifies as a true partially-ordered set; however, we can see that if we picked the elements `true` and `false` and asked *"What is their least upper bound?"*, then we can see that there isn't one.  Therefore, this set is not a join-semilattice.

### What's This Got to do with Distributed Systems?

Let's say we have a system with two replicas that hold a type of information that is very different to the shopping cart example.  Here, these replicas hold the value of our Boolean register and they then receive conflicting updates:

![Conflicting Updates](./img/L22%20Conflicting%20Updates.png)

***Q:***&nbsp;&nbsp; Why do these updates create a conflict?  
***A:***&nbsp;&nbsp; Because we have no way to combine the values `true` and `false`

***Q:***&nbsp;&nbsp; Why can we not combine these values?  
***A:***&nbsp;&nbsp; Because, as we can see from the lattice diagram above, `true` and `false` have no upper bound, let alone a least upper bound.

In this case, the inclusion set formed form the members `{empty, true, false}` contains only the members:

```
S = { {}
    , {empty}
    , {true}
    , {false}
    , {empty, true}
    , {empty, false}
    }
```

The inclusion set does not contain the least upper bound member `{true, false}` neither does it contain the upper bound `{empty, true, false}`.

In order to resolve such a conflict, we would need to implement some sort of consensus algorithm.  However, because consensus algorithms are expensive, we really don't want to implement one unless we really need to.

So generally, if the updates your replicas are receiving can be thought of as the members of a set this ***is*** a join-semilattice, then we can resolve the requirements of strong convergence by taking the least upper bound.  This also means we do ***not*** need to implement a consensus algorithm.

So, here's an informal claim:

> If the states that replicas can take on can be thought of as elements of a join-semilattice, then there is a natural way of resolving conflicts between replicas without needing a consensus algorithm.

This claim is described as *"informal"* because it uses unqualified words such as *"natural"*.  Nonetheless, there is a lot of interesting work being done on this type of conflict resolution.  If you're interested in this type of work, take a look at a topic called [*"Conflict-Free Replicated Datatypes"*](https://crdt.tech/) or (CRDTs).  An example implementation by Martin Kleppmann and Alastair Beresford has been described in this [paper](./papers/JSON%20CRDT.pdf) for JSON datatypes.

## Back to the Shopping Cart...

So far, we've been thinking about conflicts that can arise when different clients add members to a set. In the case of the shopping cart, the order in which items are added does not matter because the different members within the shopping cart have no dependency on each other.  Therefore, this situation is not one in which consensus is required.

This is a particularly useful property in the event of a network partition.  If communication is lost for some period of time between replicas, then the fact that the states of the shopping carts might diverge whilst the network partition exists does not create a problem.

As soon as the partition heals, the replicas can communicate with each other again, and their states will converge.

But what happens if we are allowed to remove items from the shopping cart?

Now things get more complicated. 

Let's say that from your laptop, you add a book to your shopping cart, and from your phone, you add a pair of jeans.

![Shopping Cart Item Deletion 1](./img/L22%20Delete%20Cart%20Item%201.png)

Both replicas synchronise and everything is fine...

From your laptop however, you've read some reviews of the book and decide that it doesn't look so interesting, so you remove it from your shopping cart &mdash; right at the very moment a network partition appears between the replicas.

![Shopping Cart Item Deletion 2](./img/L22%20Delete%20Cart%20Item%202.png)

The remove message gets through to replica 1, but not replica 2 resulting in the shopping carts being out of sync with each other.

Now from your laptop, you want to look at the contents of your shopping cart and... huh!?  That book has popped up again!

![Shopping Cart Item Deletion 3](./img/L22%20Delete%20Cart%20Item%203.png)

Maybe it's a sign that you really should read that book... or maybe it's the situation described in the Dynamo paper where, under certain circumstances, deleted items can reappear in shopping carts.

Why did this happen?

Because the contents of the shopping carts are treated as sets, and when a conflict occurs, the solution is to take the least upper bound.  With Dynamo, this resolution happens on the client, but with CRDTs, it happens in the replica.  Either way though, this approach takes the union of the sets and this can cause a deleted item to pop up again.

So how do we avoid this problem?

We could go to all trouble of ensuring that the members of your set (I.E. the contents of your shopping cart) are always the members of a join-semilattice; however, this means that you have to throw the whole cart away as soon as any item is deleted.

But wait!  There's a trick that allows us to handle this situation.  Here, we will keep track of all additions to the shopping cart in one set and all the removals from the shopping cart in a different set known as the ***Tombstone Set***.  In this case, the least upper bound of two versions of a shopping cart can be calculated simply by taking the union of the sets.


| `R1`<br>Additions | `R1`<br>Removals | `R2`<br>Additions | `R2`<br>Removals
|---|---|---|---
| 📓 | 📓 | 👖 |
| 👖 | | 📓 |

Even though replica `R2` never found out about the removal of the book, this does not matter, because that fact has been recorded in replica `R1`.  So now we can avoid having the deleted item reappear in the shopping cart by taking a two-step approach:


1. Take the union of the addition sets
1. From this, subtract the union of the removal sets

But we still haven't solved all our problems...

Let's say that even though all those bad reviews about the book caused us to delete it, we change our mind (I mean, can a book really be that bad? Let's find out).  So you add the book again.

However, look at the addition set &mdash; it already contains the book, and our addition set can only hold single instances of an item.  And the book is still in the tombstone set because we really did delete it.  So, if we left the situation like it is, even though we added the book a second time, it would disappear from the resolved shopping cart because it's in the tombstone set...

So, under these conditions, once you remove an item, it’s never coming back!

Here is where you need to do some serious design work to decide on what behaviour you want your application to have, and then think of the scenarios that could break that behaviour.

If you know that the addition of previously deleted items will be a frequently used aspect of your application's functionality, then you will need to implement some sort resolution strategy.  For instance:

* You could have of global coordination point in which all the replicas are notified that a previously deleted item is being added again and then try to ensure that everyone agrees.
* Or you could take a simplistic approach and say that additions always win over removals.
* Or you could keep a counter against each added or removed item so that adding the same book twice sets the addition counter to `2`, and removing it sets the removal counter to `-1`. Now the desired total is simply the sum of additions total and the removals total.

The bottom line is this: this is hard to get right and has been an active area of research over the last 10 years or so.  Quite a few interesting data structures have been proposed for resolving this problem, but it does not appear that any of them have been implemented in production systems yet.

As a developer however, it is difficult to reason about the data you are working with in this abstract manner. The question *"Can I really treat my data as elements of a join-semilattice?"* is difficult to answer and typically requires the help of specialist verification tools.

This is an area of research in which Lindsey Kuper is actively involved.

---

| Previous | Next
|---|---
| [Lecture 21](./Lecture%2021.md) | [Lecture 23](./Lecture%2023.md) 


---

***Endnotes***

<b id="f1">1</b>.  The name "partially ordered set" is often abbreviated to ***"poset"***

[↩](#a1)

<b id="f2">2</b>.  The "least upper bound" is known as "lub" or "join"

[↩](#a2)


