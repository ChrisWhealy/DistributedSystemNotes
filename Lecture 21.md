# Distributed Systems Lecture 21

## Lecture Given by [Lindsey Kuper](https://users.soe.ucsc.edu/~lkuper/) on May 20th, 2020 via [Twitch](https://www.twitch.tv/videos/626692883)


## Recap: MapReduce Phases

The functionality of the MapReduce framework is divided into three distinct phases:

* The ***Map*** phase
* The ***Shuffle*** phase
* The ***Reduce*** phase

### The Map Phase

This is where the developer's map function is applied to all the input data.  The data types going into and coming out of the map function are typically not the data type seen in the final output.

In general, the map function's input type is some identifiable unit of data that needs to be analysed and is supplied in the form of a key/value pair: for instance, the key could be the URL of a particular document, and the value is the document contents.

The map function then performs whatever analysis is required on that data and outputs a list of intermediate key/value pairs.  Initially, the output of the map function in written to the local storage of the map worker machine.

In the previous lecture, we used the simplified example of creating an inverted index.  Here, the input to our map function was:

```
<Doc1, <the, quick, brown, fox, jumped, over, the, lazy, dog>>
```

And the output was:

```
<the, Doc1>
<quick, Doc1>
<brown, Doc1>
<fox, Doc1>
<jumped, Doc1>
<over, Doc1>
<the, Doc1>
<lazy, Doc1>
<dog, Doc1>
```

But MapReduce is a general-purpose framework that can perform many more tasks than simply creating inverted indices.  What if we wanted to take the same document as before and count word frequency?

So, our input data is exactly the same as before:

```
<Doc1, <the, quick, brown, fox, jumped, over, the, lazy, dog>>
```

And in the simplest case, the output could be nothing more than:

```
<the, 1>
<quick, 1>
<brown, 1>
<fox, 1>
<jumped, 1>
<over, 1>
<the, 1>
<lazy, 1>
<dog, 1>
```

Here, the word `the` occurs twice in the text, but this simple case defers the addition of multiple occurrences of the same word to the reduce phase; however, should we wish to implement this optimisation in the map function, then that would be fine.  E.G.:

```
<the, 2>
<quick, 1>
<brown, 1>
...
```

Ok, but what about distributed `grep`?<sup id="a1">[1](#f1)</sup>

```
<Doc1, <the, quick, brown, fox, jumped, over, the, lazy, dog>>
<Doc2, <i, love, my, dog, spot>>
```

How the map function chooses to structure its output data is based entirely on what sort of results you want your `grep` function to produce.  For instance, let's say we want to search for the word `dog` in the above input data.  A naïve implementation of `grep` would output:

```
<Doc1, dog>
<Doc2, dog>
```

But a more sophisticated implementation may include the target word's context:

```
<Doc1, <lazy, dog>>
<Doc2, <my, dog, spot>>
```

or even the context and location offsets

```
<Doc1, <context, <lazy, dog>, offset, <7>>
<Doc2, <context, <my, dog, spot>, offset, <3>>
```

The point here is that the map function produces data in the form of key/value pairs.

Usually, these key value pairs are known as *intermediate* key/value pairs because they require further processing by the reduce function.  However, in the case of `grep`, the map function has completed the required processing by identifying where in the document the target word occurs; thus, the reduce function might simply be a ***do nothing*** function that returns whatever value it has been passed.<sup id="a2">[2](#f2)</sup>

### The Shuffle Phase

The shuffle phase is where all the intermediate key/value pairs created by the map workers is passed to the appropriate reduce workers for further processing.

But how do we decide which reduce worker is the right one?  This is decided by the partitioning function that implements some sort of hashing rule.

This partitioning function is supplied by the MapReduce framework and although Google doesn't exactly say ***how*** it has been implemented, they give the example that it could obey a rule such as `hash(key) mod N`, where `N` is the number of reduce workers.

As has already been pointed out, the `hash mod N` approach can introduce problems if the number `N` changes.  In the context of Amazon's Dynamo system, they are providing an online service that must be able to respond to unpredictable events such as sudden spikes in request volume, or hardware or network failure.  Under these circumstances, the likelihood of the number of nodes in a ring changing is high; consequently, Amazon mitigate the problems associated with changing `N` in `hash mod N` by using ***consistent hashing***.

However, in the case of Google's MapReduce, they are working in an offline environment in which the size of the input dataset is known up front; therefore, if you are the developer of the map and reduce functions, you already have the necessary information to make an informed decision about how many workers you will need.  You then use these values to configure the MapReduce framework for the expected workload during your particular batch run.

So, altering the number of reduce workers during a batch run would only happen in the event of some Byzantine error such as hardware failure or a network partition.  Under these circumstances, Google uses checkpointing for error recovery.  So, a changing value of `N` is not something you the developer need to be concerned about.


### The Reduce Phase

***Q:***&nbsp;&nbsp; What data type does the reduce function work with?  
***A:***&nbsp;&nbsp; Intermediate key/value pairs

The partitioning function provided by the MapReduce framework ensures that every key/value pair whose key hashes to the same value, is sent to the same reduce worker.  IN practical terms, the reduce function accepts a set of key/value pairs whose hashed key values fall within a certain range.

Conceptually however, the data type of the reduce function is a key to which has been bound a set of values.  Whether the set of values bound to this key is created by the partitioning function or by explicit functionality within the reduce function is not strictly important here.

In the case of the word count example, the various map functions might output intermediate key/values pairs such as:

```
<the, 2>
<quick, 1>
<brown, 1>
<fox, 1>
<jumped, 1>
<over, 1>
<lazy, 1>
<dog, 1>
```

The key values go through the `hash mod N` algorithm which then determines which reduce function will handle the next phase of the processing.  The output of a single reduce function might then be the sum of all the word count totals received from the various map workers:

```
<the, 12>
<brown, 2>
<fox, 6>
<lazy, 4>
```

What about the distributed `grep` example?

Here, each map worker either locates the search text in the document or it does not.  So, by the time we get to the reduce function, all the work has ***already*** been done.  So, the reduce function does not need to do anything other than write its input data directly to the output storage (GFS, for instance).  In fact, it is quite a common pattern for the reduce function to be little more than the identity function (see endnote [2](#f2)).


## Handling Map Worker Failure

One detail we left out of the previous discussion was the use of a ***master*** process.  This process acts as the supervisor or scheduler for the work performed by all the workers.

The master periodically pings the workers and if they do not respond within a given timeout period, the master assumes that process as failed in some way.

![MapReduce Master 1](./img/L21%20Master%201.png)

So, let's say that map worker `M1` now fails:

![MapReduce Master 2](./img/L21%20Master%202.png)

***Q:***&nbsp;&nbsp; What's happens to the work `M1` was doing?  Is it lost or can it be retrieved?  
***A:***&nbsp;&nbsp; All of `M1`'s work is lost and has to be redone

Since map workers fail from time to time, what's the best way of handling this failure?  Google had to examine the cost of the possible design options:

***Option 1)***  
Ensure that every map worker writes its intermediate key/value pairs not to its local disk that would become inaccessible in the event of failure, but to some external location from where it can be recovered

***Option 2)***  
Risk having to redo all the work assigned to a map worker if that worker fails

The answer here is simply one of time-cost &mdash; on average, which option will be quicker?

Option 1 means that the time penalty of writing data over the network must be paid for every ***successful*** run of a map worker.

However, since map workers are successful far more often than they fail, the time penalty incurred by occasionally having to redo a map worker's entire workload is offset by the fact that this will not happen too often.

This is one of the distinguishing features of MapReduce - it deliberately chooses to redo work in the event of worker failure because this is cheaper than transferring data over the network.

In general fault tolerance in distributed systems requires that we duplicate something.  We either duplicate:

* ***Data*** by storing multiple copies
* ***Communication*** by sending multiple messages
* **Effort** by redoing work


## Combine Functions

Let's look at the word count example again.  Say we want to search for the word `dog` in the string:

```
My dog spot is the best dog and the fastest dog
```

A naïve approach would be to scan the text and every time the target word is located, output an individual hit, resulting in:

```
<dog, 1>
<dog, 1>
<dog, 1>
```

But the downside of this is that three, identical intermediate key/value pairs must now be sent over the network to the reduce worker.  It would be far more efficient to derive a subtotal within the map function and then send only one intermediate key/value pair to the reduce worker.

```
<dog, 3>
```

This job is performed by a ***combine function***.  A combine function performs a task very similar to that of the reduce function, but it runs inside the map worker in order to perform local optimisation work in advance.

This is perfectly valid because the overall task we're performing is associative.  `a + b` is the same as `b + a`.

So, generally speaking, if your MapReduce task is associative, then perform as much work in the map function as possible.  This has two advantages:

* It can significantly reduce the quantity of data that needs to be transferred during the shuffle phase
* The reduce function runs much faster because (in this case) it only needs to add up subtotals, rather than the individual values discovered by the map functions.

## Map and Reduce Function Types

### Map Function

A map function takes a function and a list and applies that function to every element of the list.  So, to describe this in Haskell, the type of the map function would be written as:

```haskell
map :: (a -> b) -> [a] -> [b]
```

Breaking this down:

* `map :: (a -> a)` means that `map` takes a function that takes an input of type `a` and returns an output of type `b`
* `[a] -> [b]` describes the fact that the `map` function takes in a list of `a`'s and gives back a list of `b`'s

This function would then be implemented as follows:

```haskell
map _ [] = []
map f (x:xs) = f x : map f xs
```

So here, we have provided how `map` should behave in two situations:

```haskell
map _ [] = []
```

Here, if map is passed an empty list `[]`, then all we will do is respond with another empty list `[]`.  The underscore after `map` means that in the case of receiving an empty list, we don't care what type of function is supplied, because that function isn't going to be called anyway...

The more interesting situation is where map is passed a non-empty list:

```haskell
map f (x:xs) = f x : map f xs
```

The function passed to `map` is called `f` and the list is destructured into the variables `x` and `xs`.  `x` then contains whatever value is found at the head of the list, and `xs` contains whatever else is left (the tail - which eventually will become the empty list).

We then call function `f` passing it `x` as an argument and concatenate what we get back to the result of recursively calling `map` again passing in function `f` and whatever is left over in `xs`.

So, a simple example would be:

```haskell
map increment [1,2,3] = [2,3,4]
```

### Reduce Function

In many programming languages, the `reduce` is also known as `folder` meaning *"fold the values in the list towards the right"*.

```haskell
reduce :: (a -> b -> b) -> b -> [a] -> b
```

Breaking this down:

* `reduce -> (a -> b -> b)` means `reduce` takes is a function that takes a value of type `a` and a value of type `b` and gives back a value of type `b`.
* `b -> [a] -> b` means that it takes a value of type `b` and a list of values of type `a` and gives back a single value of type `b`

So how do we run the function passed to `reduce`?  This function needs to two arguments; a value of type `a` that comes from whatever list we're reducing, and a value of type `b`.  But what is this value of type `b`?  This value is known variously as the *"identity value"* or the *"base value"* or simply the *"accumulator"*, and acts as a starting value.

In the case that we are trying to reduce an empty list, then whatever value is supplied as the identity or the base value is simply returned unmodified.  Traditionally, the value supplied as the base case is known as the *"zero"* value, hence the `z` in the code sample below;

```haskell
reduce _ z [] = z
```

In the same way we saw above with the base case of the `map` function, the underscore here means that when `reduce` is passed an empty list, we couldn't care less about what type of function we've been supplied, because it's never going to be called anyway!  So, all we do is give back the initial zero value `z`.

In the case where `reduce` is passed a list:

```haskell
reduce f z (x:xs) = f x (reduce f z xs)
```

We call function `f` passing in the first value from the list (destructured into variable `x` which is of type `a`), but then we need a value of type `b`.  Well, we know that the `reduce` function gives us back a value of type `b`, so we recursively call `reduce` again on the tail of the list (destructured into variable `xs`) and use whatever value it returns as the required value of type `b`.

So, using this in the word count example, we would have:

```haskell
reduce add 0 [1,1,2,1,] = 5
```


<hr>
### Endnotes

<b id="f1">1</b>&nbsp;&nbsp; `grep` is often thought to be a contraction of ""***G***lobal ***Rep***lace", but the actual meaning is "***G***lobally search for a ***r***egular ***e***xpression and ***p***rint matching lines"

[↩](#a1)

<b id="f2">2</b>&nbsp;&nbsp; A function that simply returns its argument unmodified is known as the ***Identity*** function.  In JavaScript, such a function is implemented simply as 

```javascript
const id = x => x
```

[↩](#a2)


