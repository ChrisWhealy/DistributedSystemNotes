# Distributed Systems Lecture 8

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 15<sup>th</sup>, 2020 via [YouTube](https://www.youtube.com/watch?v=x1BCZ351dJk)

| Previous | Next
|---|---
| [Lecture 7](./Lecture%207.md) | [Lecture 9](./Lecture%209.md)


## Rules of the Chandy-Lamport Algorithm

This is an example of a decentralised<sup id="a1">[1](#f1)</sup> algorithm that allows you to take a global snapshot of a running distributed system.

Any process can start the snapshot without either needing to be given a special designation or the need to announce that this action is about to take place.  The act of initiating a snapshot creates a cascade of marker messages throughout the entire system that cause all the processes to take a snapshot of themselves.


### The Initiator Process

* Records its own state
* Sends a marker message out on all its outgoing channels
* Starts recording messages arriving on ***all*** incoming channels

In this case, the initiator process is `P1`.

![Chandy-Lamport Snapshot 1](./img/L8%20CL%20Snapshot%201.png)

If process `P1` decides to initiate a snapshot, then the following sequence of events takes place:

* `P1` records its own state as `S1`
* Immediately after recording its own state, `P1` sends out a marker message out on all its outgoing channels (only one in this case)
* `P1` starts recording any messages that might arrive on its incoming channels (again, only one in this case)

Notice that at the time `P1`'s snapshot happens, message `m` is currently in flight (or ***in the channel***) from `P2` to `P1`.

### Processes Receiving a Marker Message

> ***IMPORTANT***
>
> ![A Marker FIFO Anomaly Cannot Happen](./img/L8%20Marker%20FIFO%20Anomaly.png)
>
> Due the fact that all channels behave as FIFO queues, it is not possible for FIFO anomalies to occur such as a marker message arriving ***before*** an earlier event in the originating process.
>
> All of what follows would not work if we had not first eliminated the possibility of FIFO anomalies!

There are two situations here that depend on whether the process has already seen a marker message during this run of the global snapshot.

If this is the first time this process has seen a marker message, the receiver:

* Records its own state
* Flags the channel on which the marker message was received as ***empty***
* Sends out a marker message on each of its outgoing channels
* Starts recording incoming messages on all channels except the one on which it received the original marker message (now flagged as empty)

***Q:***&nbsp;&nbsp; During a snapshot, once a channel is marked as empty, what happens if you then receive a message on that channel?  
***A:***&nbsp;&nbsp; Whilst the snapshot is running, messages received on channels marked as empty are ignored!


In the diagram below, since this is the first marker message `P2` has seen, it does the following:

* It records its own state as `S2`
* Marks channel <code>C<sub>12</sub></code> as empty
* Sends out a marker message on all its outgoing channels (in this case, only channel <code>C<sub>21</sub></code>)
* Normally, it would now start recording any messages that arrive on its other, incoming channels, but in this case, since its only incoming channel (<code>C<sub>12</sub></code>) has already been marked as empty, there is nothing to record

![Chandy-Lamport Snapshot 2](./img/L8%20CL%20Snapshot%202.png)

However, if a process ***has*** already seen a marker message during this run of the global snapshot (sending out a marker message counts as "seeing" a marker), then for the channel on which the marker message arrived, the receiver:

* Stops recording incoming messages on that channel
* Sets that channel's final state to be the sequence of all messages received whilst recording was active

In the case of `P2`'s marker message arriving at `P1`, `P1` has already "seen" a marker message because it sent one out in the first place.  

Message `m` from `P2` (sent at event `C`) arrives on channel <code>C<sub>21</sub></code> as event `D` on process `P1`.  This message arrived ***before*** the marker message because channels always behave as FIFO queues.

Upon receiving this marker message, `P1` does the following:

* It stops recording on the marker message's channel (<code>C<sub>21</sub></code>)
* The state of channel <code>C<sub>21</sub></code> is set to the sequence of messages that arrived whilst recording was active

![Chandy-Lamport Snapshot 3](./img/L8%20CL%20Snapshot%203.png)

So, we now have a complete snapshot of our entire system, which in this simple case, consists of four things:

1. The state of our two processes:
    * `P1`'s state recorded as `S1`
    * `P2`'s state recorded as `S2`
1. The state of all channels between those processes:
    * Channel <code>C<sub>12</sub></code> recorded by `P2` (Empty)
    * Channel <code>C<sub>21</sub></code> recorded by `P1` (Message `m`)


## The Chandy-Lamport Algorithm in a More Detailed Scenario

When a snapshot takes place, every process ends up sending out a marker message to every other process.  So, for a system containing `N` participating processes, `N * (N - 1)` marker messages will be sent.

This might seem inefficient as the number of messages rises quadratically with the number of participating processes, but unfortunately, there is no better approach.

As stated in the previous lecture, the success of the Chandy-Lamport algorithm relies entirely on the truth of the following assumptions:

1. Eventual message delivery is guaranteed, thus making delivery failure impossible
1. All channels act as FIFO queues, thus eliminating the possibility of messages being delivered out of order
1. Processes don't crash! (See [lecture 10](./Lecture%2010.md))

### A Worked Example

In this example, we have three communicating processes `P1`, `P2` and `P3` in our system, and we want to take a snapshot.

Process `P1` acts as the initiator; so, it follows the above steps:

* It records its own state as <code>S<sub>1</sub></code>
* It sends out two marker messages; one to `P2` and one to `P3` - but notice that the arrival of the marker message at `P2` is delayed.  
    This turns out not to be a problem.
* `P1` starts recording on both its incoming channels <code>C<sub>21</sub></code> and <code>C<sub>31</sub></code>

![Chandy-Lamport Example Step 1](./img/L8%20Snapshot%20Ex%201.png)

Next, `P3` receives the marker message from `P1`.  Since this is the first marker message it has received:

* It records its own state as `S3`
* Marks the channel on which it received the marker message (<code>C<sub>13</sub></code>) as empty
* Sends out marker messages on all its outgoing channels
* Starts recording on its other incoming channel (<code>C<sub>23</sub></code>)

![Chandy-Lamport Example Step 2](./img/L8%20Snapshot%20Ex%202.png)

Looking at `P3`'s marker message that now arrives at `P1`, since `P1` initiated the snapshot process, this not the first marker it has seen, so `P1`:

* Stops recording incoming messages on that channel (<code>C<sub>31</sub></code>)
* Sets that channel's final state to be the sequence of all messages received whilst recording was active - which is none - so the channel state of <code>C<sub>31</sub></code> is `{}`.

![Chandy-Lamport Example Step 3](./img/L8%20Snapshot%20Ex%203.png)

Now looking at the other marker message from `P3` to `P2`. This is the first marker `P2` has seen, so it:

* It records its own state as `S2`
* Marks the channel on which it received the marker message (<code>C<sub>32</sub></code>) as empty
* Sends out marker messages on all its outgoing channels
* Starts recording on its other incoming channel (<code>C<sub>12</sub></code>)

![Chandy-Lamport Example Step 4](./img/L8%20Snapshot%20Ex%204.png)

Eventually, the initial marker message from `P1` arrives at `P2`.  This is the second marker `P2` has seen, so it:

* Stops recording incoming messages on that channel (<code>C<sub>12</sub></code>)
* Sets that channel's final state to be the sequence of all messages received whilst recording was active - which is none - so the channel state of <code>C<sub>12</sub></code> is `{}`.

![Chandy-Lamport Example Step 5](./img/L8%20Snapshot%20Ex%205.png)

`P2`'s marker message now arrives at `P1`.  This is not the first marker `P1` has seen, so it:

* Stops recording incoming messages on that channel (<code>C<sub>21</sub></code>)
* Sets that channel's final state to be the sequence of all messages received whilst recording was active - which in this case is the message `m3` sent at event `H` in `P2` to event `D` in `P1` - so the channel state of <code>C<sub>12</sub></code> is `{m3}`.

![Chandy-Lamport Example Step 6](./img/L8%20Snapshot%20Ex%206.png)

Lastly, the marker message from `P2` arrives at `P3`.  This is not the first marker `P3` has seen, so it:

* Stops recording incoming messages on that channel (<code>C<sub>23</sub></code>)
* Sets that channel's final state to be the sequence of all messages received whilst recording was active - which is none - so the channel state of <code>C<sub>23</sub></code> is `{}`.

![Chandy-Lamport Example Step 7](./img/L8%20Snapshot%20Ex%207.png)

We now have a coherent snapshot of the entire system composed of three process states:

* <code>P1 = S<sub>1</sub></code>
* <code>P2 = S<sub>2</sub></code>
* <code>P3 = S<sub>3</sub></code>

And six channel states:

* <code>C<sub>12</sub> = {}</code>
* <code>C<sub>21</sub> = {m3}</code>
* <code>C<sub>13</sub> = {}</code>
* <code>C<sub>31</sub> = {}</code>
* <code>C<sub>23</sub> = {}</code>
* <code>C<sub>32</sub> = {}</code>

### What About the Internal Events Not Recorded in the Process Snapshots?

In the above diagram, events `C`, `D` and `E` do not form part of `P1`'s snapshot recorded in state <code>S<sub>1</sub></code> because these events had not yet occurred at the time `P1` decided to take its snapshot.

Similarly, events `J` and `K` do not form part of `P3`'s snapshot recorded in state <code>S<sub>3</sub></code> because these events had not yet occurred at the time the marker message arrived from `P1`.

These events will all be recorded the next time a snapshot is taken.


## How Does the Entire System Know When the Snapshot Is Complete?

An individual process knows its local snapshot is complete when it has recorded:

* Its own internal state, and
* The state of ***all*** its incoming channels

If it can be shown that the snapshot process terminates for an individual process, then it follows that it will terminate for all participating processes in the system.

Now we can appreciate the importance of the assumptions listed at the start.  The success of this entire algorithm rests on the fact that:

* Eventual message delivery is guaranteed, and
* Messages never arrive out of order (all channels are FIFO queues), and
* Processes do not crash (yeah, right! Again, see [lecture 10](./Lecture%2010.md))

In Chandy & Lamport's [original paper](https://lamport.azurewebsites.net/pubs/chandy.pdf) they provide a proof that the snapshot process does in fact terminate.

Determining the snapshot for the entire system however lies outside the rules of the Chandy-Lamport algorithm and needs to be handled by some external coordinator that stitches all the individual process snapshots together.



---

| Previous | Next
|---|---
| [Lecture 7](./Lecture%207.md) | [Lecture 9](./Lecture%209.md)




---

***Endnotes***

<b id="f1">1</b>&nbsp;&nbsp; In this context, a "decentralised algorithm" is one that does not need to be invoked from a special coordinating process; any process in the system can act as the initiator.  A beneficial side-effect of this is that if two processes simultaneously decide to initiate a snapshot, then nothing bad happens.

[↩](#a1)

