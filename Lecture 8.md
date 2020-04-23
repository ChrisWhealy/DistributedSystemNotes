# Distributed Systems Lecture 8

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on April 15th, 2020 via [YouTube](https://www.youtube.com/watch?v=x1BCZ351dJk)

## Rules of the Chandy-Lamport Algorithm

This is an example of a decentralised<sup id="a1">[1](#f1)</sup> algorithm that allows you to take a global snapshot of a running distributed system.

Any process can start the snapshot without either needing to be given a special designation or need to announce that this action is about to take place.  Any process can initiate a snapshot and in doing so, causes a cascade of message throughout the entire system that result in all other processes taking a snapshot of themselves.


### The Initiator Process

* Records its own state
* Sends a marker message out on all its outgoing channels
* Starts recording messages arriving on ***all*** incoming messages

In this case, the initiator process is `P1`.

![Chandy Lamport Snapshot 1](./img/L8%20CL%20Snapshot%201.png)

`P1` records its own state as `S1`, immediately sends a marker message out on all its outgoing channels (only one in this case) and then starts recording any messages that might arrive on its incoming channels (again, only one in this case).

Notice that at the time `P1`'s snapshot happens, a message is currently in flight (or ***in the channel***) from `P2` to `P1`.

### Receiver of a Marker Message

> ***IMPORTANT***
>
> ![A Marker FIFO Anomaly Cannot Happen](./img/L8%20Marker%20FIFO%20Anomaly.png)
>
> Due the fact that all channels behave as FIFO queues, it is not possible for FIFO anomalies to occur such as a marker message arriving ***before*** an earlier event in the originating process.
>
> All of what follows would not work if we had not first eliminated the possibility of FIFO anomalies!

There are two cases depending on whether the process has seen a marker message before during this run of the global snapshot.

If this is the first time this process has seen a marker message, the receiver:

* Records its own state
* Flags the channel on which the marker message was received as ***empty***
* Sends out a marker message on all its outgoing channels
* Starts recording incoming messages on all channels except the one on which it received the original marker message (now flagged as empty)

> ***Question***  
> During a snapshot, once a channel is marked as empty, what happens if you then receive a message on that channel?
> 
> ***Answer***  
> Whilst the snapshot is running, messages received on channels marked as empty are ignored!


In the diagram below, since this is the first marker message `P2` has seen, it does the following:

* It records its own state as `S2`
* Marks channel <code>C<sub>12</sub></code> as empty
* Sends out a marker message on all its outgoing channels (in this case, only channel <code>C<sub>21</sub></code>)
* Normally, it would now start recording any messages that arrive on its other, incoming channels, but in this case, since its only incoming channel (<code>C<sub>12</sub></code>) has already been marked as empty, there is nothing to record

![Chandy Lamport Snapshot 2](./img/L8%20CL%20Snapshot%202.png)

However, if a process ***has*** already seen a marker message during this run of the global snapshot (sending out a marker message counts as "seeing" a marker), then for the channel on which the marker message arrived, the receiver:

* Stops recording incoming messages on that channel
* Sets that channel's final state to be the sequence of all messages received whilst recording was active

In the case of `P2`'s marker message arriving at `P1`, `P1` has already "seen" a marker message because it sent one out in the first place.  

Message `m` from `P2` (sent at event `C`) arrives on channel <code>C<sub>21</sub></code> as event `D` on process `P1`.  This message arrived ***before*** the marker message because channels always behave as FIFO queues.

Upon receiving this marker message, `P1` does the following:

* It stops recording on the marker message's channel (<code>C<sub>21</sub></code>)
* The state of channel <code>C<sub>21</sub></code> is set to the sequence of messages that arrived whilst recording was active

![Chandy Lamport Snapshot 3](./img/L8%20CL%20Snapshot%203.png)

So, we now have a complete snapshot of our entire system, which in this simple case, consists of four things:

1. The state of our two processes:
    * `P1`'s state recorded as `S1`
    * `P2`'s state recorded as `S2`
1. The state of all channels between those processes:
    * Channel <code>C<sub>12</sub></code> recorded by `P2` (Empty)
    * Channel <code>C<sub>21</sub></code> recorded by `P1` (Message `m`)


## The Chandy-Lamport Algorithm in a More Detailed Scenario

Snapshot takes place, in general every process ends up sending out a marker message to every other process.  So, for a system containing `N` participating processes `N * (N - 1)` marker messages will be sent.

As stated in the previous lecture notes, the success of the Chandy-Lamport algorithm relies entirely on the truth of the following assumptions:

1. Eventual message delivery is ***guaranteed***, thus making delivery failure impossible
1. All channels act as FIFO queues, thus making it impossible for messages to be delivered out of order (I.E. it guarantees that there will never be any FIFO anomalies)
1. Processes don't crash! (The topic of process failure is dealt with in detail in the next lecture)

### A Worked Example

In this example, we have three communicating processes `P1`, `P2` and `P3` in our system, and we need a snapshot.

Process `P1` acts as the initiator; so, it follows the above steps:

* It records its own state as <code>S<sub>1</sub></code>
* It sends out two marker messages to `P2` and `P3` - but notice that the arrival of this marker message at `P2` is delayed.  This turns out not to be a problem.
* `P1` starts recording on both its incoming channels <code>C<sub>21</sub></code> and <code>C<sub>31</sub></code>

![Chandy-Lamport Example Step 1](./img/L8%20Snapshot%20Ex%201.png)

Next, `P3` receives the marker message from `P1`.  Since this is the first marker message it has received:

* It records its own state as `S3`
* Marks the channel on which it received the marker message (<code>C<sub>13</sub></code>) as empty
* Sends out marker messages on all its outgoing channels
* Starts recording on its other incoming channel (<code>C<sub>23</sub></code>)

![Chandy-Lamport Example Step 2](./img/L8%20Snapshot%20Ex%202.png)

Looking at the marker message that now arrives at `P1`, since `P1` initiated the snapshot process, this not the first marker to arrive, so `P1`:

* Stops recording incoming messages on that channel (<code>C<sub>31</sub></code>)
* Sets that channel's final state to be the sequence of all messages received whilst recording was active - which is none - so the channel state of <code>C<sub>31</sub></code> is `{}`.

![Chandy-Lamport Example Step 3](./img/L8%20Snapshot%20Ex%203.png)

Now looking at the other marker message from `P3` to `P2`. This is the first marker `P2` has seen, so it:

* It records its own state as `S2`
* Marks the channel on which it received the marker message (<code>C<sub>32</sub></code>) as empty
* Sends out marker messages on all its outgoing channels
* Starts recording on its other incoming channel (<code>C<sub>12</sub></code>)

![Chandy-Lamport Example Step 4](./img/L8%20Snapshot%20Ex%204.png)

Eventually, the marker initial message from `P1` arrives at `P2`.  This is the second marker `P2` has seen, so it:

* Stops recording incoming messages on that channel (<code>C<sub>12</sub></code>)
* Sets that channel's final state to be the sequence of all messages received whilst recording was active - which is none - so the channel state of <code>C<sub>12</sub></code> is `{}`.

![Chandy-Lamport Example Step 5](./img/L8%20Snapshot%20Ex%205.png)

`P2`'s marker message now arrives at `P1`.  This is not the first marker `P1` has seen, so it:

* Stops recording incoming messages on that channel (<code>C<sub>21</sub></code>)
* Sets that channel's final state to be the sequence of all messages received whilst recording was active - which in this case is the message `m3` the was sent at event `H` in `P2` to event `D` in `P1` - so the channel state of <code>C<sub>12</sub></code> is `{m3}`.

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

An individual process knows its local snapshot is complete when:

* It has recorded its own internal state
* It has recorded the state of ***all*** its incoming channels

If it can be shown that the snapshot process terminates for an individual process, then it follows that it will terminate for all participating processes in the system.

Now we can appreciate the importance of the validity of the assumptions listed above: that of eventual message delivery, and all channels act as FIFO queues (this preventing message arriving out of order).

So, the success of this entire algorithm rests on the fact that all messages will eventually be delivered in the order they were sent.

In Chandy & Lamport's original paper they provide a proof that the snapshot process does in fact terminate.

Determining the snapshot for the entire system however lies outside the rules of the Chandy-Lamport algorithm and needs to be handled by some external process that stitches all the individual process snapshots together.







<hr>
<b id="f1">1</b> In this context, a "decentralised algorithm" is one that does not need to be invoked from a special coordinating process; any process in the system can act as the initiator.  A beneficial side-effect of this is that if two processes simultaneously decide to initiate a snapshot, then nothing detrimental happens.[↩](#a1)
