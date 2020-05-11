# Distributed Systems

Notes on [YouTube lectures](https://www.youtube.com/user/lindseykuper/videos) given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/), Assistant Professor of Computing as UCSC

Please excuse cases where I might have given a poor summary of something Lindsey was saying &mdash; I'm a student here...

There are no notes for lecture 1 as this was concerned with course administration and logistics.

|  Date | Description | Subjects Recapped |
|---|---|---|
| [Lecture 2](./Lecture%202.md)<br>April 1<sup>st</sup>, 2020 | Distributed Systems: What and why?<br>Time and clocks |
| [Lecture 3](./Lecture%203.md)<br>April 3<sup>rd</sup>, 2020| Lamport diagrams<br>Causality and happens-before<br>Network models<br>State and events<br>Partial orders
| [Lecture 4](./Lecture%204.md)<br>April 6<sup>th</sup>, 2020 | Total orders and Lamport clocks | Partial orders<br>Happens-before
| [Lecture 5](./Lecture%205.md)<br>April 8<sup>th</sup>, 2020 | Vector clocks<br>Protocol runs and anomalies<br>Delivery vs. Receiving<br>FIFO delivery | Lamport Clocks
| [Lecture 6](./Lecture%206.md)<br>April 10<sup>th</sup>, 2020 | Causal delivery<br>Totally-ordered delivery<br>Implementing FIFO delivery<br>Preview of implementing causal broadcast | Delivery vs. receiving<br>FIFO delivery
| [Lecture 7](./Lecture%207.md)<br>April 13<sup>th</sup>, 2020 | Implementing causal broadcast<br>Uses of causality in distributed systems<br>Consistent snapshots<br>Preview of Chandy-Lamport snapshot algorithm | Causal anomalies and vector clocks
| [Lecture 8](./Lecture%208.md)<br>April 15<sup>th</sup>, 2020 | Chandy-Lamport Snapshot Algorithm | 
| [Lecture 9](./Lecture%209.md)<br>April 17<sup>th</sup>, 2020 | Chandy-Lamport wrap-up: limitations, assumptions and properties<br>Uses of snapshots<br>Centralized vs. decentralized algorithms<br>Safety and liveness | Delivery guarantees and protocols
| [Lecture 10](./Lecture%2010.md)<br>April 20<sup>th</sup>, 2020 | Reliable delivery<br>Fault classification and fault models<br>The Two Generals problem | Safety and liveness
| [Lecture 11](./Lecture%2011.md)<br>April 22<sup>nd</sup>, 2020 | Implementing reliable delivery<br> Idempotence<br>At-least-once/at-most-once/exactly-once delivery<br>Unicast/Broadcast/Multicast<br>Reliable broadcast<br>Implementing reliable broadcast<br>Preview of replication
| [Lecture 12](./Lecture%2012.md)<br>April 24<sup>th</sup>, 2020 | Replication<br>Total order vs. determinism<br>Consistency models: FIFO, causal and strong<br>Primary-backup replication<br> Chain replication<br>Latency and throughput 
| [Lecture 13](./Lecture%2013.md)<br>April 27<sup>th</sup>, 2020 | **Pause for breath!**<br>Wrapping up replication techniques
| [Lecture 14](./Lecture%2014.md)<br>May 1<sup>st</sup>, 2020 | Handling node failure in replication protocols<br>Introduction to consensus<br>Problems equivalent to consensus<br>The FLP result<br>Introduction to Paxos | Strongly consistent replication protocols
| [Lecture 15](./Lecture%2015.md)<br>May 4<sup>th</sup>, 2020 | Paxos: the interesting parts
| [Lecture 16](./Lecture%2016.md)<br>May 6<sup>th</sup>, 2020 | Paxos wrap-up: Non-termination, Multi-Paxos, Fault tolerance<br>Other consensus protocols: Viewstamped Replication, Zab, Raft<br>Passive vs. Active (state machine) replication


