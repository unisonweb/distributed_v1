# RFC: Unison distributed programming API

_This document was written by Heather Miller and Paul Chiusano, but the ideas developed here are the work of many other contributors too, including Arya Irani, Rúnar Bjarnason, Chris Gibbs, Dan Doel, and others. Thanks to everyone who has helped out in big and small ways!_

__Disclaimer: This document is a work in progress. Comments are welcome!__

This document describes a distributed programming API for Unison. The API is intended to be general purpose, capable of expressing protocol implementations, orchestrations of serverless computations, data-centric/MapReduce-style computations, and more. A goal is to support this generality without having to futz with sockets or worry about networking layers; the developer should get to focus on the essence of the distributed algorithm, data structure, or protocol being implemented, not on parsing or serialization or low-level networking.

The API is given using [abilities](https://unisonweb.org/docs/abilities), which leaves it abstract and capable of being interpreted in different ways. The same program may be run locally on a single machine, or via a _distributed runtime_ that executes the program on distributed infrastructure.

## Remote

Unison's main API for writing distributed programs is the `Remote` ability, which supports forking parallel computations at different logical locations. Forking returns a handle to a running task, and `await t` blocks the current computation until the task `t` completes with its result. The Unison runtime has lightweight threads that are decoupled from OS threads, so many Unison threads may be mapped by the runtime to the same OS thread.

The `forkAt`/`await` API is much the same as other asynchronous programming APIs, except that tasks now have a location (locations are essentially GUIDs, more on that below). Here's the full API, which will be explained in more detail:

```Haskell
unique ability Remote task g where
  forkAt   : Location -> '{g, Remote task g} a -> task a
  detachAt : Location -> '{g, Remote task g} a -> task a
  location : task a -> Location
  await    : task a -> a
  cancel   : Failure -> task a -> ()
  fail     : Failure -> a
  here     : Location
```

The `Remote` ability is parameterized on `task`, which is the representation of a running computation, and `g` which controls what abilities forked computations can access. For example:

* A computation `'{Remote task, {}} Nat` can only do pure computation.
* A computation `'{Remote task, Http.Client} ()` will have access to an `Http.Client` ability.
* In theory, a `'{Remote task, IO} a` could exist in which remote computations can do arbitrary `IO`, but in practice, on real distributed infrastructure, we'll use more restricted abilities than `IO`.

The primary function that provides for distributed parallelism is `forkAt`, which starts a computation running at a (possibly remote) location. Here's an example usage:

```haskell
-- Example program
parallelAdd loc t1 t2 =
  a = forkAt loc t1
  b = forkAt loc t2
  await a + await b
```

### Locations and location chaining

The `Location` type is just a GUID representing a logical location. These will be mapped to physical machines by the distributed runtime. Multiple `Location` values may map to the same physical machine, or a `Location` may refer to multiple physical machines. For instance, a `usEast : Location` might refer to an entire region, consisting of thousands of physical machines, and `forkAt usEast myTask` gives the distributed runtime the freedom to schedule `myTask` onto any available machine in that region.

Note that `forkAt` returns a `task` with possibly more fine-grained location information, which can be used to achieve better task locality and reduce communication. For instance, in this code:

```Haskell
t1 = forkAt usEast thing1
t2 = forkAt usEast thing2
merged = forkAt usEast '(merge (await t1) (await t2))
```

The `t1`, `t2`, and `merged` tasks might all be scheduled onto different machines, leading to network communication when their results are being combined with `merge`. Instead, the programmer can request that tasks be forked _at the same location as other tasks_, like so:

```Haskell
t1 = forkAt usEast thing1
t2 = forkAt (location t1) thing2
merged = forkAt (location t1) '(merge (await t1) (await t2))
```

Here, `t2` and `merged` are both forked at the same location as `t1`, so no network communication happens to combine their results.

<details><summary>
The above API lets you be explicit about where tasks are forked. One idea that we have discussed is the notion of an <i>algebra of locations</i> that makes it possible to be less explicit about locations while still expressing interesting constraints.</summary>
<br/>
For instance, it might be desirable to be able to say things like <code>forkAt (somewhereNear bob) thing1</code>, or <code>forkAt (both (within usEast) (farFrom bob)) thing2</code> for a location that's in `usEast` but isn't on the same machine as the location `bob`. This sort of thing might be used to obtain locations whose failures will be less correlated than if they are on the same machine. We don't know if an algebra of locations would be too rich/flexible, effectively making it overkill, or not. This is an idea worth exploring further.
</details>

### Error handling

Tasks (especially remote tasks) can fail. We use a common failure type with runtime type tags so users can define new kinds of failure:

```haskell
unique type Failure = { tag : Link.Type, msg : Text, payload : Any }

unique ability Remote task g where
  fail : Failure -> x
  ...
```

The `fail` operation allows failures to be triggered explicitly from user code. For example, here we define a new failure type and a convenience function for raising errors of that type within `Remote`:

```Haskell
unique type LoginFailure = -- empty, just using the type for its tag

loginFailure : Text -> a ->{Remote task g} x
loginFailure msg payload = Remote.fail (Failure (typeLink LoginFailure) msg (Any payload))
```

Various error handling functions can be written generically, for instance:

```Haskell
-- Catch ALL errors
try : '{Remote task g} a ->{Remote task g} Either Failure a
try r = ...

-- Catch only errors of a particular type
tryType : Link.Type -> '{Remote task g} a ->{Remote task g} Either Failure a
tryType t r = match try r with
  Left e -> if tag e == t then Left e else fail e
  Right a -> a

-- Catch all errors and resume using a function `onErr`
rescue : '{Remote task g} a
      -> (Failure ->{Remote task g} a)
      ->{Remote task g} a
rescue r onErr = match try r with
  Left err -> onErr err
  Right a  -> a
```

In addition to user defined failure types, the interpreter of `Remote` can inject infrastructure failures (like a networking failure, out of memory errors, and so on). Here's a sketch of a few kinds of errors that the distributed runtime might inject:

```haskell
-- Just empty types used as a tag
unique type Timeout =

-- The location couldn't be contacted (likely due to a networking error)
unique type Unreachable =

-- The task was explicitly cancelled
unique type Cancelled =

-- The task ran out of memory before completing
unique type OutOfMemory =

-- The task couldn't be placed at the hinted location because that location
-- was full
unique type LocationFull =

-- etc
```

### Task trees, cancellation, and detaching

```Haskell
unique ability Remote task g where
  fail : Failure -> x
  cancel : Failure -> task a -> ()
  ...
```

Tasks form a tree: a forked task can fork subtasks. When a task completes or is cancelled, all its subtasks will be cancelled if they aren't already completed. For example, in this code:

```Haskell
ex1 : Nat ->{Remote task Http} task Nat
ex1 n = forkAt usEast 'let
  t1 = forkAt usEast '(thing1 n "https://example.com")
  t2 = forkAt usEast '(thing2 n "https://google.com")
  await t2
```

The `t1` task will be actively cancelled (if it isn't already completed) when the `ex1` task completes (or is cancelled).

A separate function, `detachAt` has the same signature as `forkAt` but creates a task with no parent. This is sometimes useful when starting a task purely for its effects (rather than its result).

```Haskell
detachAt : Location -> '{g, Remote task g} a -> task a
```

## Augmenting `Remote` with other abilities: storage, I/O, message passing

`Remote` by itself just provides task parallelism, an old and well-understood parallelization strategy. By selecting different abilities for `g` in `'{Remote task g} a`, we can describe more interesting distributed programs that use durable or ephemeral storage layers, do message passing or various forms of I/O (like issuing HTTP requests to load data from S3, say), or any combination of abilities inside the parallel tasks.

As we will see in the [Examples](#examples) section later on in this document:

* `Remote` with storage abilities can be used for distributed map-reduce or batch computing interfaces like RDDs in Spark.
* `Remote` with a message passing ability can be used to implement distributed algorithms like Paxos, Raft, gossip protocols, and so on.

### Durable storage

Stable remote data can be represented in Unison using the `Durable` ability. `Durable`s are what they sound like--remote data that is expected to always be reachable.

The API for the read and write portions of `Durable`s are as follows:

```haskell
unique type Durable a = { hash : Hash, location : Location }

-- Discussed below
unique type RefreshRate = { seconds : Nat, backoffLimit : Optional Nat }

unique ability Durable.W where
  save : a -> Durable a

  -- Data not accessed at least this often may be deleted by storage layer
  -- or moved to archival storage. See later discussion on GC.
  refreshRate : RefreshRate

unique ability Durable.R where
  restore : Durable a -> a

  -- Refresh a reference without loading it. Returns `false` if the reference
  -- can't be found in the storage layer. Rarely used but sometimes useful.
  -- See GC discussion below.
  refresh : Durable a -> Boolean
```

At runtime, a `Durable a` is represented as a `Hash` and a `Location` where the value is stored.

Here, the notion of *location* once again comes up in the API, and we can use chaining to provide hints to the distributed runtime about where to initially place or cache durable data. Note that since `Durable` values are immutable and content addressed, they can be replicated to more locations beyond the location originally hinted by the programmer. Different distributed runtimes could make different choices about this.

```Haskell
t = forkAt cluster1 '(save dataset1) -- saves somewhere in cluster1
forkAt (Durable.location (await t)) '(save dataset2) -- saves at the task `t`
```

There's discussion of garbage collection of `Durable`s below.

#### Distributed immutable data structures

Using `Durable`, one can implement distributed immutable data structures. These work much like in-memory data structures, but with regular pointers replaced with `Durable` pointers. Because these data structures are just references to immutable data stored externally, they are lightweight and can be passed between distributed tasks spawned within a `Remote` computation.

To show the basic idea of constructing these data structures, here's a distributed immutable sequence type, and a function `index` for random access.

```Haskell
unique type Seq d a
  = Empty
  | One a
  | Two Nat (d (Seq d a)) (d (Seq d a))

size : Seq d a -> Nat
size = cases
  Empty -> 0
  One _ -> 1
  Two n _ _ -> n

index : Nat -> Seq Durable a ->{Durable.R} Optional a
index n = cases
  Empty -> None
  One a -> if n == 0 then Some a else None
  Two m l r ->
    loadedL = restore l
    if n < size loadedL then index n loadedL
    else index (n `drop` size loadedL) (restore r)
```

The implementation of `index` is straightforward and looks almost identical to what the code would be if all the data were in memory. The difference is just some calls to `restore` in a few places.

This implementation also shows how operations on these data structures can be memory efficient as well. There's no need to load "the whole data structure" into memory to implement `index` or many other operations. Assuming the tree is balanced, only a logarithmic number of nodes in the tree get loaded into memory before locating the requested index.

### Ephemeral storage

While a `Durable` is expected to always be reachable, an `Ephemeral` is not. `Ephemeral`s are meant to be used for intermediate remote data that is not important to cache or store long-term. It would typically be stored in RAM and swapped to durable storage only if needed.

The API is basically identical to `Durable`:

```haskell
unique type Ephemeral a = { hash : Hash, location : Location }

unique ability Ephemeral.W where
  save     : a -> Ephemeral a

  -- Data not accessed at least this often may be deleted by storage layer
  -- or moved to archival storage. See later discussion on GC.
  refreshRate : RefreshRate

unique ability Ephemeral.R where
  restore : Ephemeral a -> a

  -- Refresh a reference without loading it. Returns `false` if the reference
  -- can't be found in the storage layer. Rarely used by sometimes useful.
  refresh : Durable a -> Boolean
```

This can be used for distributed data types much like `Durable`. For instance, `Seq Ephemeral a` might be used to represent an ephemeral data set spread across the RAM of a few thousand machines.

Some question we are actively discussing/working out:

* Again the same questions come up around garbage collection of `Ephemeral`, but they are even more pressing because `Ephemeral` will get used for temporary data structures, like heap memory in single machine programs.
* Possible fault tolerance of `Ephemeral` values.

### Garbage collection of `Durable` and `Ephemeral`

In a distributed setting, heartbeats are often used for garbage collection: a node using some external resource keeps that resource alive by sending the node(s) hosting the resource regular hearbeats. If the user node is hit by an asteroid, it stops sending heartbeats, and if there are no other users of the resource sending heartbeats, the resource knows to delete itself. (Explicit deallocation or referencing counting is problematic, because the node using the resource can get hit by an asteroid before it gets to deallocate or decrement the reference count.)

While this concept is straighforward, and `Durable` and `Ephemeral` references ("storage references") each have a `refreshRate` operation to control how often data should be refreshed or read before deallocation, implementing it naively could involve lots of network communication. These storage references are extremely fine-grained: data structures like distributed sequences, maps, and so on will often be composed of trees of storage references with potentially trillions of branches. We want this flexibility of the storage API being capable of expressing any data structure, but it would be inefficient if every node in a large data structure is sending separate heartbeats to the storage layer.

The solution involves a few different pieces:

* If a storage reference is live, each of its dependencies should be considered live as well. This avoids needing to traverse an entire huge data structure to send heartbeats for all its constituent storage references. Instead, merely keeping the root of the structure live is enough to retain all its descendents.
* Reading an `Ephemeral` or a `Durable` storage reference counts as a heartbeat. This avoids needing separate network communication apart from the code that reads the reference, which will often already involve network communication if the data isn't cached.
* The distributed runtime can employ exponential backoff of refresh rate, so that long-lived storage references only require a logarithmic number of heartbeats.
  * For instance, an initial refresh rate of every 10 minutes may be increased to 20 minutes, 40 minutes, and so on, up to some (optional) limit. The reduces the amount of network communication especially when storage references can be found in a local cache.
  * This approach means deallocation may be less timely than otherwise. For instance, if the refresh rate is doubled as part of the backoff process, a storage reference that is no longer live as of time _t_ may kept around until time _2t_. So for things like GDPR-compliant storage where there is a [Right to Erasure](https://www.threatstack.com/blog/gdpr-what-is-the-right-to-erasure#:~:text=Under%20Article%2012.3%20of%20the,the%20complexity%20of%20the%20request.) that requires that certain data be deleted within 30 days, the user may allow for some backoff of the refresh rate but supply a limit of 30 days.

Some additional miscellaneous notes about this API:

* It's possible for a distributed runtime to establish other GC roots besides just storage references that have been accessed recently. For instance, the system might have a general concept of a running job and consider the output of a job to be a GC root, and any transitive dependencies of that output may be considered live by a runtime.
* It's possible to use this API to create lexically scoped references that are guaranteed to remain alive for the duration of a task and be deallocated when some lexical scope finishes. The general idea is spawn a subtask that issues periodic keepalives (with an initially small refresh rate, like every 20 seconds) to the referenced data, and then have that subtask terminate when the overall task completes.
  * We can even use the typechecker to that such references don't escape some lexical scope (see [this comment on how](https://github.com/unisonweb/unison/issues/798#issuecomment-702487151) which is a similar idea to the `ST` type in Haskell).

### Message-passing layer

Implementing protocols like Paxos or other distributed algorithms requires explicit messaging and coordination. For that reason, Unison will have a messaging layer that is accessible when needed. This API is rather low level and imperative, and we expect higher-level APIs will be built so developers aren't always programming with message passing directly.

Like `Durable` and `Ephemeral`, this is another ability that can be used by `Remote` computations: a `'{Remote task {Messaging, Ephemeral.R}} Nat` is a distributed computation that uses message passing and reads from ephemeral storage.

Here's the API:

```haskell
unique type Channel a = { id : GUID, location : Location }

unique ability Messaging where
  -- create a channel at the current location, for instance, in:
  -- channelAtBob = forkAt bob 'channel
  channel    : Channel a

  -- Send a value to a channel; works anywhere. If the channel
  -- resides on another node, will serialize the `a` and send it
  -- over the network.
  send       : a -> Channel a -> ()

  -- Like `send`, but don't wait for the value to be transferred
  -- before returning.
  sendUnconfirmed : a -> Channel a -> ()

  -- Receive a value from a channel; works anywhere. If the channel
  -- resides on another node, the receive will happen on that node
  -- and the result sent to the caller of `receive`.
  receive    : Channel a -> a

  -- Like `receive` but leave the value in the channel.
  peek       : Channel a -> a

  -- Like `receive`/`peek` but if the channel is currently empty,
  -- returns `None` right away rather than blocking.
  receiveNow : Channel a -> Optional a
  peekNow    : Channel a -> Optional a
```

## Distributed Runtime

Unison's distributed programming facilities cannot be described by APIs and type signatures alone. Rather, there are several behind-the-scenes aspects of a running distributed Unison program that can only be managed by a robust set of background processes that form a distributed runtime.

Aspects such as identity in a running cluster (where exactly is the node identified by some identifier `x`?), scheduling, propagating error information, and management of failure-handling strategies are all runtime affairs.

The Distributed Unison runtime is meant to be a series of running processes on each node in a Unison cluster.

### Scheduling

How does the Unison Distributed runtime know *which* nodes are most appropriate to schedule tasks on in some pool of nodes?

The Unison Distributed runtime is responsible for understanding the topology of all the locations it has access to. As the Unison Distribute runtime is a running process on all Unison nodes, it keeps track of which node it is running on, and the location of its neighbors. Using pings, it may even be possible to work out which nodes are near (or, quicker to reach) or far (or, slower to reach).

With such bookkeeping in place, it would then be possible to expose an API for this notion of near vs far (quickness/slowness to reply), and programmers could write libraries that can take advantage of locality. This would effectively give programmers a hook to avoid stragglers for latency-sensitive tasks. And if there is ever to be an algebra of locations, this near vs far (quick vs slow) could even be worked into the algebra of locations.

## Examples

We're planning on showing some short examples of using these APIs to program a variety of distributed programs. Some ideas for examples:

* Distributed map reduce
* A Spark-like library
* Simple ping-pong message passing example
* Toy implementation of Paxos
* Distributed mutex
* Gossip protocol

In a subsequent document, we intend to work out some or all of these examples to help illustrate how these APIs could be used for different sorts of applications.
