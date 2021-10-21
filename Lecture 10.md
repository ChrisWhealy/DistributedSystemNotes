# Distributed Systems Lecture 10

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 20<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=5QH37qLyO7A)

| Previous | Next
|---|---
| [Lecture 9](./Lecture%2009.md) | [Lecture 11](./Lecture%2011.md)

## Recap: Safety & Liveness Properties

| Safety Property | Liveness Property
|---|---|
| Something bad will ***not*** happen | Something good ***eventually*** happens
| Can be violated in a finite execution.<br>![FIFO Anomaly](./img/L5%20FIFO%20Anomaly.png) | Cannot be violated in finite execution.<br>![No violation - yet](./img/L5%20Protocol%204.png)
| Examples of safety properties include all the delivery guarantees we've spoken about so far such as FIFO, Causal and Totally-Ordered | An example of a liveness property is the guarantee that the system will ***eventually*** respond to a client request.<br>However, the diagram above is not a counterexample because the definitions of liveness guarantees tend to be open-ended and therefore cannot exclude the possibility of *"waiting forever"*

The trouble with Distributed System design is that in order to be useful, it needs both types of property to be present; however, the open-ended nature of liveness properties makes them hard to reason about.

Let's say we want to implement a protocol that satisfies the safety property of FIFO delivery, but doesn't need to care about any liveness properties.
So, we could build a system that either drops or ignores every message...

![Vacuous FIFO Delivery](./img/L10%20Vacuous%20FIFO%20Delivery.png)

This is completely useless, but at least it guarantees FIFO delivery! (Doh!)

So, it turns out that on their own, neither safety nor liveness properties are of much practical use.
We need a system that implements a combination of both.

### Liveness Property: Reliable Delivery

Other than the *"eventual deliver"* property we've spoken of, the only other liveness property we've mentioned is *"reliable delivery"*, and the definition varies depending on who you ask, but here's one definition:

> Let `P1` be a process that sends a message `m` to process `P2`.  
> If neither `P1` nor `P2` crashes, then `P2` eventually delivers message `m`

However, you might also see an extended version of the above definition:

> Let `P1` be a process that sends a message `m` to process `P2`.  
> If neither `P1` nor `P2` crashes ***and not all messages are lost***, then `P2` eventually delivers message `m`

These definitions vary only in respect of whether or not we should be concerned about message loss.
This question then leads us into the topic of fault models that we will come to shortly.
However, here we assert that reliable delivery is a liveness property.

But why is this?

Firstly, we can see that the idea of reliable delivery cannot be violated in a finite execution: that is, in a finite execution, we cannot draw a Lamport diagram that demonstrates a violating counter-example.

So, since we know that all the properties we care about are either safety properties, liveness properties or some combination of both, and we know that *"reliable delivery"* is not a safety property, we know it must therefore be a liveness property.

## So, What Exactly is a "Fault"?

Any time you design a system, you need to make certain assumptions about the environment in which that system will be operating.
So, before you can describe your system as being ***fault tolerant***, you must first clearly define what circumstances could occur that constitute a fault, and secondly, how your system will respond under such circumstances.
Only then will you understand what conditions your system needs to tolerate.

In other words, we must start with a clear understanding of what exactly constitutes a fault.

In a simple scenario like the one below, machine `M1` asks machine `M2` the question *"What's the value of `x`?"* and expects to hear something back such as `x=5`.

![Possible Faults](./img/L10%20Possible%20Faults.png)

What sort of faults could occur here?

* `M2` never receives the message because it was lost due to a communication fault
* `M2` gets the message, but is unable to answer it because it became corrupted
* `M2` gets the message, but only after a significant delay
* `M2` does not respond because it has already crashed
* `M2` crashes as a result of trying to answer `M1`'s question
* `M2` is too busy to answer the question
* `M2` ignores the message
* `M2` deliberately responds with the wrong answer
* `M2` sends a response back to `M1` which gets lost, corrupted or delayed
* `M2` sends a response back to `M1`, but then `M1` crashes as it tries to proces the answer
* etc...

The problem here is that even in this minimal scenario, we cannot enumerate all the possible errors the might occur; however, we can categorise them.

### Fault Categories

Let's arrange these types of fault into different informal categories.

| Fault Category | Description
|---|---
| Crash | Software execution halts or hardware fails
| Omission | Messages are sent, but not received (I.E. lost)
| Timing | Messages are sent and processed successfully, but very slowly<br>Also known as a ***performance*** fault
| Byzantine<sup id="a1">[1](#f1)</sup> | Intentionally malicious or arbitrary behaviour

These fault categories have been used in Distributed System's design since about the early 1990's and are presented above in hierarchical order, starting with the lowest level first.

### Crash Fault

At the bottom of this hierarchy is the ***crash fault***.
This simply means that a process has stopped exchanging messages with the other participants in the system.

A crash fault can happen for a variety of reasons: for instance, execution could halt due to a software failure, or the process might continue to handle its own internal messages but cease responding to external messages.
Whilst this second case is not due to software execution halting, as far as the other participants in the system are concerned, since this process takes no further part in the overall operation of the system, it may as well have crashed.

### Omission Fault

When a process continues execution, but for whatever reason, fails to send or receive some, but not all messages.

### Timing (or Performance) Fault

A process responds either too late or too early.
The typical case is where a process responds too slowly; however, at the hardware level, it is possible for a response to be too fast.

In this course, we're going to sidestep the discussion of timing faults because we will be focussing on the asynchronous message delivery model &mdash; which never makes any delivery time guarantees in the first place!

### Byzantine Fault

A process behaves in an arbitrarily erroneous, or possibly even malicious way.
Usually, such behaviour is intentional.
Again, in this course we won't be spending much time talking about this type of fault.

## Fault Hierarchy

Why are the fault categories listed in this particular order?

Let's says we have two different protocols, `X` and `Y`.

| | Protocol `X` | Protocol `Y`
|---|---|---|
| **Tolerates** | Crash Faults | Omission Faults

If Protocol `Y` tolerates omission faults, does it also tolerate crash faults?
If so, why?

### Crash and Omission Faults

If a process crashes, then ***every*** message sent to that process will not be received; therefore, this is a special case of an omission fault in which a process fails to send or receive only a certain number of messages.

![Fault Hierarchy 1](./img/L10%20Fault%20Hierarchy%201.png)

Hence if process `P1` can tolerate process `P2` failing to send or receive ***some*** messages (an omission fault), then by definition, `P1` must also be able to tolerate the case where `P2` fails to send or receive ***all*** messages (a crash fault).

The set of crash faults form a proper subset within the set of omission faults.

### Timing Faults

In the same way that crash faults are a special case of omission faults, omission faults are a special case of timing faults.

A timing fault is where the response to a message is received outside an acceptable timeframe (usually, too slowly), and an omission fault is where the response to a message is never received (infinitely too slow)

![Fault Hierarchy 2](./img/L10%20Fault%20Hierarchy%202.png)

### Byzantine Faults

A Byzantine fault is where a process behaves in an arbitrary or intentionally malicious way.
So, if a protocol is tolerant of Byzantine faults, does this mean it must also tolerate timing faults?

Yes, it does.

The reason is that if a process `B` (for Byzantine) starts behaving in an arbitrary or malicious manner, all other processes communicating with `B` will be unable to distinguish faults in `B` due to timing errors, with faults in `B` due to arbitrary or malicious behaviour.

Consequently, if process `B` decides that to say:

* *"I'm going to pretend to crash"*, or
* *"I'm not going to respond to certain messages from certain other processes"*, or
* *I'm deliberately going to mislead two processes by swapping my responses around*

Any process communicating with `B` will be unable to determine the true cause of such behaviour.

> ***My Aside***
> 
> This is because computers are unable of determine the intent of, or motive behind an action, their inferences can only be made on the mechanistic basis of cause and effect

The list of possible fault behaviours here is endless...

![Fault Hierarchy 3](./img/L10%20Fault%20Hierarchy%203.png)

For more details, see the paper ["Atomic Broadcast: From Simple Message Diffusion to Byzantine Agreement"](./papers/atomic_broadcast.pdf) by Cristian et al.

Given the fact that we will confine our discussion to systems that use asynchronous communication, such systems cannot give any guarantees about message delivery times; therefore, for the remainder of this course, discussion around fault categories will look more like this:

![Fault Hierarchy 4](./img/L10%20Fault%20Hierarchy%204.png)

## Are These the Only Fault Categories?

No: these fault categories can be subdivided.
For instance, some Byzantine faults are easier to deal with than others.

For instance, a Byzantine Fault that is relatively easy to deal with is message alteration or duplication.

If you receive a message that has been altered (maliciously or otherwise), then such faults are generally fairly easy to detect using techniques such as checksums or message hashes.
What's much harder to detect is message corruption where that corruption is so subtle that your authentication technique fails to detect it.<sup id="a2">[2](#f2)</sup>

So, we can redraw the above diagram as follows:

![Fault Hierarchy 5](./img/L10%20Fault%20Hierarchy%205.png)

If we detect that a message has been corrupted, the simplest way of handling it is to ignore it completely.
In this case, we have handled the message in the same way as if we had never received it in the first place.
Thus, we have downgraded this particular type of Byzantine fault to an Omission fault. 

## Fault Models

Based on the preceding discussion we can define a ***Fault Model*** as:

> The specification of faults that a system may exhibit, which in turn defines the kind of faults which will be tolerated by the system.

Fault models are nested in the same way as fault categories.

In general, the easiest faults to tolerate are the ones in the centre of the diagram.
As we move outwards, fault model implementation becomes increasingly complicated simply because a greater number and type of bad things can happen.

In this class, we will be looking mostly at the Omission Model.
This will still be hard to implement however, because there are still plenty of things that can go wrong!

## The Two Generals Problem

This is a classic thought experiment that dates back to 1975 from the appendix of a paper by Akkoyunlu, Ekanadham and Huber called "[Some Constraints and Trade-offs in the Design of Network Communications](./papers/net_comms_constraints_tradeoffs.pdf)"

In this situation, two armies commanded by Generals Alice and Bob want to attack a common enemy; however, several significant challenges must be overcome before victory can be assured:

* Alice's and Bob's armies are camped in two valleys on opposites sides of their common enemy
* The enemy is camped in the valley in between Alice and Bob
* The enemy has sufficient strength that victory cannot be assured unless Alice and Bob attack simultaneously
* Alice and Bob can only communicate with each other by sending messengers through enemy territory
* Alice and Bob cannot be sure that any messages they send will get through because the messenger risks being captured, and the message fails to get through

![Two Generals 1](./img/L10%20Two%20Generals.png)

What form of communication can these Generals have with each other so that they can both be confident that they are acting on some shared, common knowledge?
For instance, if the plan is to attack tomorrow at dawn, then how can both Generals be sure that the other has firstly agreed to this plan, and secondly, is ready to act on it?

Since the armies of Alice and Bob are not strong enough on their own to defeat this enemy, victory is only possible if they arrive at a commonly agreed plan of action

Let's look at a few scenarios:

* Alice sends Bob the message ***"Attack at dawn!"***.  

    ***Q:***&nbsp;&nbsp;Having sent this message, should Alice simply go ahead and attack at dawn?  
    ***A:***&nbsp;&nbsp;No, because she has no way of knowing that Bob has even received the message, let alone agreed to it.  

    If Alice attacks on her own, there is a significant chance of defeat, so she should wait for Bob's confirmation that he has agreed to the plan.

* Let's now say that Bob has received the message, agreed to it, and sent a confirmation back saying *"Yes, we attack at dawn!"*.  

    ***Q:***&nbsp;&nbsp;Should Bob now go ahead and attack at dawn?  
    ***A:***&nbsp;&nbsp;No!  Because he does not know that Alice has received his confirmation &mdash; again, a lone attack would be very risky.

* Let's then say that Alice receives Bob's agreement to attack at dawn - is this good enough for Alice to launch an attack?  No, because Bob doesn't know that Alice knows that he's agreed to attack...  

    And this is all getting very silly and sounds like good material for a Monty Python sketch!

As it turns out, it has been proven impossible to eliminate 100% of the uncertainty inherent in systems where communication reliability cannot be guaranteed.

### Indeterminacy of the Omission Model

In the Omission Model, it is impossible for either of the two Generals to know with 100% certainty that the other has agreed to attack.
This example is just one of many fundamental impossibility results in distributed systems, and it applies in any setting where the communication between participants is liable to failure.

## Work Arounds for the Two Generals Problem

Some possibilities do exist here:

* Instead of trying to decide on the basis of unreliable communication, make a plan in advance and share that plan between the participants.
  In other words, act on the basis of shared common knowledge<sup id="a3">[3](#f3)</sup>.
* Since unreliable communication excludes the possibility of achieving ***total*** certainty, the best we can hope for is ***reasonable*** certainty.
 
To achieve reasonable certainty, we could adopt the following strategy:

1. Alice could send Bob a sequence of messages knowing that on average, at least one will get through.
1. For every one of Alice's messages that successfully reaches Bob, Bob sends back an acknowledgement.
   Again, since not of all these acknowledgments will make it back to Alice, multiple acknowledgements are needed in the hope that at least one will get through.
1. Alice receives Bob's first acknowledgement and stops sending her stream of messages.
1. After a certain period of time, Bob's confidence will start to grow that at least one of his acknowledgements got through because Alice has stopped sending her original message.

The objective here is not to remove all uncertainty, but to lower it to a level considered acceptable enough to act upon.

---

| Previous | Next
|---|---
| [Lecture 9](./Lecture%2009.md) | [Lecture 11](./Lecture%2011.md)


---
***Endnotes***

<b id="f1">1</b>&nbsp;&nbsp; Why is this type of fault called a <b><i>"Byzantine"</i></b> fault?

The name comes from an example in a paper by Leslie Lamport, Robert Shostak and Marshall Pease.  Here, they describe a situation in which, in order to avoid being defeated by their enemy, the different Generals in the Byzantine Army must all agree on a coordinated strategy for capturing a city; however, it is suspected that some of these Generals might be traitors and will thus attempt to disrupt the plans of the loyal generals.

To avoid any political issues that might have been created by choosing Generals and Armies from the recent past, they chose to use an example from the [Byzantine Empire](https://en.wikipedia.org/wiki/Byzantine_Empire) &mdash; the eastern successor empire to the Romans after their empire fell in the 5th Century AD.  The original paper can be found [here](./papers/byzantine.pdf)

The point here is that the behaviour of a small number of malicious participants will not jeopardise the overall success of the system.

[↩](#a1)

<b id="f2">2</b>&nbsp;&nbsp; Discussion of such authentication techniques is not covered in this course

[↩](#a2)

<b id="f3">3</b>&nbsp;&nbsp; Knowledge is said to be "common" when all participants in a system know with 100% certainty that all other participants share the same knowledge.  But not only is this very difficult to establish externally, it is also very fragile.  What would happen for instance, if someone needs to change the plan?  How could the changed plan then become accepted, common knowledge?

[↩](#a3)
