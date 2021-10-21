# Distributed Systems

Lecture notes for course [CSE138, Spring 2020](http://composition.al/CSE138-2020-03/index.html) given by [Prof Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/), Assistant Professor of Computing at UCSC

Due to the Covid-19 lockdown being enforced at the time, these lectures had to be delivered online and are available on [YouTube](https://www.youtube.com/user/lindseykuper/videos) and [Twitch](https://www.twitch.tv/lindseykuper/videos)

This series of lectures also includes a discussion panel with recent grad students and two guest lecturers.
Notes have not been created for these events; however, you can watch the videos here:

* ["Grad Student Discussion Panel"](https://www.youtube.com/watch?v=ArKapXZkJvM) Lindsey Kuper talks with Emma Gomish, Pete Wilcox and Lawrence Lawson.
   May 15<sup>th</sup>, 2020
* ["Blockchain Consensus"](https://www.youtube.com/watch?v=m6qZY7_ingY) by Chris Colohan, May 27<sup>th</sup>, 2020
* ["Building Peer-to-Peer Applications"](https://www.twitch.tv/videos/640120840) by Karissa McKelvey, June 3<sup>rd</sup>, 2020

|  Date | Description | Subjects Recapped |
|---|---|---|
| Lecture 1 | There are no notes for this lecture as it was concerned with course administration and logistics  |
| [Lecture 2](./Lecture%2002.md)<br>April&nbsp;1<sup>st</sup>,&nbsp;2020 | Distributed Systems: What and why?<br>Time and clocks |
| [Lecture 3](./Lecture%2003.md)<br>April&nbsp;3<sup>rd</sup>,&nbsp;2020| Lamport diagrams<br>Causality and the "happens before" relation<br>Network models<br>State and events<br>Partial orders
| [Lecture 4](./Lecture%2004.md)<br>April&nbsp;6<sup>th</sup>,&nbsp;2020 | Total orders and Lamport clocks | Partial orders<br>Happens before relation
| [Lecture 5](./Lecture%2005.md)<br>April&nbsp;8<sup>th</sup>,&nbsp;2020 | Vector clocks<br>Protocol runs and anomalies<br>Message Delivery vs. Message Receipt<br>FIFO delivery | Lamport Clocks
| [Lecture 6](./Lecture%2006.md)<br>April&nbsp;10<sup>th</sup>,&nbsp;2020 | Causal delivery<br>Totally-ordered delivery<br>Implementing FIFO delivery<br>Preview of implementing causal broadcast | Delivery vs. Receipt<br>FIFO delivery
| [Lecture 7](./Lecture%2007.md)<br>April&nbsp;13<sup>th</sup>,&nbsp;2020 | Implementing causal broadcast<br>Uses of causality in distributed systems<br>Consistent snapshots<br>Preview of the Chandy-Lamport snapshot algorithm | Causal anomalies and vector clocks
| [Lecture 8](./Lecture%2008.md)<br>April&nbsp;15<sup>th</sup>,&nbsp;2020 | Chandy-Lamport Snapshot Algorithm |
| [Lecture 9](./Lecture%2009.md)<br>April&nbsp;17<sup>th</sup>,&nbsp;2020 | Chandy-Lamport wrap-up: limitations, assumptions and properties<br>Uses of snapshots<br>Centralized vs. decentralized algorithms<br>Safety and liveness | Delivery guarantees and protocols
| [Lecture 10](./Lecture%2010.md)<br>April&nbsp;20<sup>th</sup>,&nbsp;2020 | Reliable delivery<br>Fault classification and fault models<br>The Two Generals problem | Safety and liveness
| [Lecture 11](./Lecture%2011.md)<br>April&nbsp;22<sup>nd</sup>,&nbsp;2020 | Implementing reliable delivery<br> Idempotence<br>At-least-once/at-most-once/exactly-once delivery<br>Unicast/Broadcast/Multicast<br>Reliable broadcast<br>Implementing reliable broadcast<br>Preview of replication
| [Lecture 12](./Lecture%2012.md)<br>April&nbsp;24<sup>th</sup>,&nbsp;2020 | Replication<br>Total order vs. determinism<br>Consistency models: FIFO, causal and strong<br>Primary-backup replication<br> Chain replication<br>Latency and throughput
| [Lecture 13](./Lecture%2013.md)<br>April&nbsp;27<sup>th</sup>,&nbsp;2020 | **Pause for breath!**<br>Wrapping up replication techniques
| [Lecture 14](./Lecture%2014.md)<br>May&nbsp;1<sup>st</sup>,&nbsp;2020 | Handling node failure in replication protocols<br>Introduction to consensus<br>Problems equivalent to consensus<br>The FLP result<br>Introduction to Paxos | Strongly consistent replication protocols
| [Lecture 15](./Lecture%2015.md)<br>May&nbsp;4<sup>th</sup>,&nbsp;2020 | Paxos: the interesting parts
| [Lecture 16](./Lecture%2016.md)<br>May&nbsp;6<sup>th</sup>,&nbsp;2020 | Paxos wrap-up: Non-termination, Multi-Paxos, Fault tolerance<br>Other consensus protocols: Viewstamped Replication, Zab, Raft<br>Passive vs. Active (state machine) replication
| [Lecture 17](./Lecture%2017.md)<br>May&nbsp;8<sup>th</sup>,&nbsp;2020 | Eventual consistency<br>Strong convergence and strong eventual consistency<br>Introduction to application-specific conflict resolution<br>Network partitions<br>Availability<br>The consistency/availability trade-off
| [Lecture 18](./Lecture%2018.md)<br>May&nbsp;11<sup>th</sup>,&nbsp;2020 | Dynamo: A review of old ideas<ul><li>Availability</li><li>Network partitions</li><li>Eventual consistency</li><li>Vector clocks</li><li>Application-specific conflict resolution</li></ul>Introduction to:<ul><li>Anti-entropy with Merkle trees</li><li>Gossip</li><li>Quorum consistency</li></ul>
| [Lecture 19](./Lecture%2019.md)<br>May&nbsp;13<sup>th</sup>,&nbsp;2020 | More about Quorum Consistency<br>Introduction to sharding<br>Consistent hashing
| [Lecture 20](./Lecture%2020.md)<br>May&nbsp;18<sup>th</sup>,&nbsp;2020 | Online systems vs. Offline systems<br>Raw data vs. Derived data<br>Introduction to Google's MapReduce framework<br>MapReduce example: transform a forward index into an inverted index
| [Lecture 21](./Lecture%2021.md)<br>May&nbsp;20<sup>th</sup>,&nbsp;2020 | MapReduce<ul><li>Types</li><li>Approach to fault tolerance</li><li>Combine functions</li><li>More examples</li></ul> | MapReduce phases
| [Lecture 22](./Lecture%2022.md)<br>May&nbsp;29<sup>th</sup>,&nbsp;2020 | The math behind replica conflict resolution<ul><li>Upper bounds</li><li>Least upper bounds</li><li>Join-semilattices</li></ul> | Strong convergence<br>Partial orders
| [Lecture 23](./Lecture%2023.md)<br>June&nbsp;1<sup>st</sup>,&nbsp;2020 | Filling in the gaps: Overviews of 2-phase commit and Practical Byzantine Fault Tolerance (PBFT)<br>Quick overview of the history of:<ul><li>Lamport and Vector Clocks</li><li>Replication Strategies</li><li>Consensus</li><li>Replication Needs Consensus</li></ul>
