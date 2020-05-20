# Distributed Systems Lecture 20

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 18th, 2020 via [Twitch](https://www.twitch.tv/videos/624791387)

## How is Google's [MapReduce Paper](./papers/MapReduce.pdf) Related to Distributed Systems?

### Online Systems

Up until now, we have focused on the design of systems to which you make requests, and from which you receive responses.  Key/value stores such as Amazon's Dynamo work this way; Web servers also work this way, as do databases and proxy caches.

In the case of Web Servers, you should not think only in terms of general-purpose HTTP servers such as Apache or Nginx, but also in terms of apps that have a built-in, dedicated HTTP server as the primary means for interaction.

All of these systems can be lumped together into the category of "Services" or "Online Systems".  Such systems wait for incoming client requests and then they try to handle them as quickly as possible (given the constraints of whatever consistency model they're implementing).  These systems typically prioritise:

* Low latency
* High availability

Under these circumstances, response times are typically in the range of a few tens to a few hundreds of milliseconds.


### Offline (Batch Processing) Systems

However, there are other types of system that take in huge amounts of data in order to analyse it in some way.  Typical examples of such analysis include restructuring user data to make it more useful to a specific user group, or to derive some form of summary.

The goal here is not to achieve low latency, but high throughput - the faster you can crunch your way through one terabyte of data, the better.  Consequently, the time taken to perform this type of distributed analysis is often significant - of the order of hundreds or even thousands of seconds.

So, the goal of an offline system is to achieve a very high degree of parallelism, as opposed to the high degree of concurrency needed by an online system.

> ***ASIDE***  
> The concepts of parallelism and concurrency are often confused.  They are distinct, but related concepts that are often implemented together within the same system.
> 
> * ***Concurrency***  
>     Provides a high degree of scalability and fault tolerance by allowing multiple (typically small) tasks to be handled simultaneously  
> * ***Parallelism***  
>     Provides raw speed by allowing a single (typically very large) task to be broken down into a large number of small, independent subtasks that can all be executed simultaneously.

Google's MapReduce system and its successors are examples of offline systems that solve problems by means of high degrees of parallelism.

It should also be noted that Google published their MapReduce paper in 2004 and since then, this technology has been superseded by more generic tools.  As of 2014, Google stopped using MapReduce as their data analytics tool in favour of [Cloud Dataflow](https://www.datacenterknowledge.com/archives/2014/06/25/google-dumps-mapreduce-favor-new-hyper-scale-analytics-system/)

## Your Sharding Strategy Depends on Your Access Strategy

Up until now, we have only looked at data that is stored as simple key/value pairs and then accessed by key lookup.  However, as soon as you change either your data model or your access strategy, then the question of how your data should be sharded becomes a lot more subtle.

So, instead of having a simple key/value store, what if you had a relational database with a detailed schema?  Now, the way you shard your data would be based on the type of queries you expect.

For example, if you are Facebook end-user, then you access your data in one way, but if you are a Facebook data scientist analysing social graphs, then you need to access the same data, but in a very different way.  Here is a situation in which two user groups have very different perspectives on the same data, thus they require very different patterns of access.

As far as performance is concerned, FaceBook's priority is to ensure that their end-users experience the fastest possible response time.  But the priority of minimum end-user response time does not help the FaceBook data scientists trying to analyse how the website is being accessed, or how social graphs develop and change over time.  Therefore, in order for the both the end-users to enjoy a fast response time, and for the data scientists to be able to do their job efficiently, multiple copies of the data are needed.  You could argue that having multiple copies of the data is redundant, but the different structures used by the different datastores make a huge difference to the time taken for the different user groups to perform their work.

Google's MapReduce paper gives these two types of data different names:

* ***Raw data***  
    The authoritative version of the data
* ***Derived data***  
    One of more copies of the raw data that have been processed and restructured in some way (for instance, some metrics have been applied in order to derive a summary of some sort).  
    Derived data can also be created from other sets of derived data.

Google's MapReduce is a tool for computing derived data.

### Building an Inverted Index

In order to make the Web searchable, you need an index that allows people to search for a Web page by specifying one or more keywords.  But the Web is not organised by keywords, it is organised first into websites, then each website contains multiple pages (or documents).

Google tackles this problem first by using crawlers that go out and gulp down entire websites.  Then, all these documents are stored *as-is* in what is known as a ***forward index***.

| Document | Words
|---|---
| Doc 1 | the, quick, brown, fox...
| Doc 2 | the, dog, growls, at, the, fox...

Here, the data is organised by document.  If you know the name of the document, you can discover all the words within that document.  But this is not how people search the web: people know what words they're searching for, but don't know which documents contain those words.  In other words, we need to create the style of index that is found at the back of a book, where you look up a word in order to discover the pages where that word is used.

There's nothing wrong with storing the data in the form of a forward index, it’s just that in order to be useful to people wanting to search the Web, this cannot be the ***only*** structure used to store the data.

If we restructure this data, we can produce a list sorted by words, from which we can then discover the documents containing those words.

| Word | Found in Documents
|---|---
| at | Doc 2
| brown | Doc 1
| dog | Doc 2
| fox | Doc 1, Doc 2
| growls | Doc 2
| quick | Doc 1
| the | Doc 1, Doc 2

Compared to the forward index, we have not changed the amount of information in the inverted index.  It’s exactly the same information, only it’s been rearranged in such a way that makes a different sort of lookup very easy.

So, what sort of processing is needed to transform a forward index into an inverted index?

To answer this question, we don't actually need to know anything about distributed systems.  This is a simple algorithmic question that can be answered by breaking the problem down into a couple of steps:

1. For every document in the forward index, scan that document and for every word encountered, emit an intermediate key/value pair containing the word and the name of the document.
1. Sort all the word/document pairs by word, collating the document names in which that word occurs

![Transform a Forward Index into an Inverted Index](./img/L20%20Inverted%20Index.png)

Now we have inverted the index and made documents discoverable on the basis of the words they contain (whereas previously, the words were only discoverable if you knew the document in which they lived)

Conceptually, this is not a complicated process&hellip;

***Q:***&nbsp;&nbsp; So why are we even talking about this in a distributed systems class?  
***A:***&nbsp;&nbsp; Because when you want to perform this task on a huge dataset in as short a time as possible, it becomes a distributed systems problem

Now, in order to solve this problem on a massive scale, we need to bring in everything we've learnt about scalability, consistency and fault tolerance.

## MapReduce Functionality from the Inside

The basic observation made by Google was that although the transformations that need to be performed on the data are, in themselves, quite straight forward, that simplicity is obscured by the complexity of having to perform this task in a highly distributed, scalable and fault tolerant manner.  Therefore, wouldn't it be good if there were an easy-to-use framework that hid all the distributed systems complexity and simply allowed developers to represent their problem in a generalised manner?

This is what MapReduce was built to provide.

Due to the enormous size of the datasets involved here, there is no way that either the forward or the inverted index would ever be able to fit on a single machine, so immediately, we have to make some very important data sharding decisions.

Using the example of inverting a list of Web pages into a word index, the fact that the data making up the forward index is already split up across a large number of machines is not a problem.  Each machine can still contribute towards the overall solution by creating a set of intermediate key/value pairs from whatever subset of documents it has in its possession.  Since each document is processed in exactly the same way, it doesn't in fact matter whether an individual machine possess a large or small number of documents.  Also, since there are no dependencies between documents, the order in which they are processed is not important.

Here is an example of how a very large task can be split into a large number of independent subtasks, and then each of those subtasks executed simultaneously.  This type of parallelism is known as ***embarrassingly parallel***.   In other words, it is so easy to split up the task into parallel subtasks, that it would be embarrassing not to.

Some tasks are very hard to parallelize, but this is not one of them.

For the sake of simplicity, let's imagine that every document in the forward index is allocated to a single machine, and that each machine builds its own set of intermediate key/value pairs:

![Distributed MapReduce 1](./img/L20%20Distributed%20MapReduce%201.png)

It's important to point out that so far, no network communication has been required.  All the intermediate key/value pairs have been written to each machine's local hard disk.  This is what allows each `M` machine to work independently of any other `M` machine.

Well that's nice, but so far, we're only halfway to a solution.  These intermediate key/value pairs now need to be collated to form the forward index.

Let's now say that we have two machines for the second phase of the computation (<code>R<sub>1</sub></code> and <code>R<sub>2</sub></code>).  So how would we decide which machine should handle which key?

We've answered this question before when we looked at consistent hashing.  If we take the hash value of the key in the intermediate key/value pair and `mod` it by the number of available `R` machines, then we can guarantee that the same words will always be collated by the same machine.

![Distributed MapReduce 2](./img/L20%20Distributed%20MapReduce%202.png)

Several questions remain however:

1. Didn't we say in the last lecture that using `hash mod N` is not a good strategy because if `N` changes, then you have to move all your data around?
1. How did the data get from machines <code>M<sub>1</sub></code>, <code>M<sub>2</sub></code> and <code>M<sub>3</sub></code> over to <code>R<sub>1</sub></code> and <code>R<sub>2</sub></code>?
1. What happens to the output of the `R` machines - where does that go?

### Doesn't `hash mod N` Introduce Problems if `N` Changes?

In the case of Amazon Dynamo, a naïve implementation of `hash mod N` will certainly lead to problems if `N` changes.  This is because you are working in an online environment in which it is very difficult (impossible?) to predict the expected data volume; therefore, you must be able to scale rapidly (I.E. add new nodes, thus changing `N`) in order to handle spikes in request volume.  Contrast this however with Google's MapReduce environment, and you find you're working in an offline situation where you already know exactly how much data your current batch will be processing.  Therefore, since you know you're working is a stable environment, you can scale the number of `M` and `R` machines accordingly.

There is, of course, the possibility that a machine could fail.  So, MapReduce uses a check-pointing strategy that allows for a new machine to take over the computation from the failed machine's last check point.  That way, only a minimal amount of reworking is needed in order to continue with forward progress.

### How is the Data Moved Between the Machines?

The transfer of data from the `M` machines to the `R` machines is known as the "shuffle".  This is quite a time-consuming phase because all the data accumulated on each of the `M` machines needs to be transferred over the network to the respective `R` machines &mdash; and as Google point out in their paper, network bandwidth is a rare commodity.

![Distributed MapReduce 3](./img/L20%20Distributed%20MapReduce%203.png)

### What Happens to the Output of the `R` Machines?

Once the various `R` machines have collated their portion of the data, the MapReduce framework handles writing that data to Google's distributed file system GFS.

> ***ASIDE***  
> Google no longer use the GFS distributed file system, but since this is referenced in the paper, it is included here. 

### MapReduce as a Framework
 
So, we can identify three distinct phases to this batch process:

* The ***Map*** phase
* The ***Shuffle*** phase
* The ***Reduce*** phase

![Distributed MapReduce 4](./img/L20%20Distributed%20MapReduce%204.png)


Each machine involved in the Map phase is sent a small program that, in this case, executes the following pseudo-code:

```
for each word W in document D {
  emit WordDocumentPair(W,D)
}
```

This program is known as a ***Map Function*** because the action passing every element in a list as an argument to a function is known as *"mapping"*.  The function is said to have been *mapped* across the elements of the list.

Similarly, the machines involved in the reduce phase perform an equally simple task implemented in the ***Reduce Function***.  The reduce function takes the intermediate key/value pairs and first sorts them by key, then collates all the different values for the same key into a single list.


So Google's MapReduce system is framework into which you insert your map and reduce functions, then the framework handles all the ugliness of distributing your functionality across thousands of machines, handling all the communication, check-pointing and fault tolerance, and finally passing the data between the map, shuffle and reduce phases to generate the required output.

In addition to providing your map and reduce functions, you need to configure the MapReduce framework to tell it:

* How many workers you want allocated to the map phase? (E.G. 200,000)
* How many workers you want allocated to the reduce phase? (E.G. 5,000)
* On how many physical machines should all of those workers run? (E.G. 2,000)

There is a popular Open Source clone of Google's MapReduce known as [Hadoop](https://en.wikipedia.org/wiki/Apache_Hadoop) which includes its own clone of GFS known as HDFS (Hadoop Distributed Filesystem)

### Questions

***Q:***&nbsp;&nbsp; Is generating an inverted index what MapReduce is mainly used for?  
***A:***&nbsp;&nbsp; The use case of the inverted index is the canonical example used to illustrate the general applicability of this technique to a wide range of problems.  MapReduce allows you to plug in your own map and reduce functions that are typically (but not exclusively) used to process large volumes of text data.

## Examples of Problems Solved Using MapReduce

Google's paper gives a variety of different problems that can all be solved using the MapReduce programming model.

* Word count
* Inverted index
* Distributed `grep`
* Distributed sort
* etc...

As long as you can find a way to express the solution to your problem in terms of a map function and reduce function, then the MapReduce framework will be able to help you.  The key points here are:

* You must be able to split your input data into units that have no dependencies between them (E.G. the individual documents retrieved by a crawler from a set of web sites)
* You must be able to write a map function that can be applied ***in any order*** to these individual units of data
* The identity of the relevant reduce worker is determined when the intermediate key/value pairs are passed through the hash algorithm during the shuffle phase.
* The output of the reduce function(s) is/are the final analysed data from this run of the MapReduce framework

Additionally, you might need to provide certain configuration parameters for the number of map and reduce workers you expect to need, but in the end, the MapReduce framework implements all the plumbing code, leaving a couple of gaps into which you insert your own code.








