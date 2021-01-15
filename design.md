*This is a design document that is a work in progress.* 

An early intention of Unison was for it to facilitate the design and implementation of distributed applications. Unison intends to be a language that makes it easier to implement and reason about programs as diverse as protocol implementations, orchestrations of serverless computations, data-centric/MapReduce-style computations, and more.

Importantly, as a programming language meant to be used to implement features of distributed applications, Unison attempts a useful separation of concerns for developers of distributed systems. An important design consideration is thus to allow the distributed system developer to reason about important aspects of distributed system development like communication patterns over the network, but without having to dip to lower levels of the OSI model. Distributed applications written in Unison are meant to sit at the presentation/application level of the OSI model, without having to dip lower. That is, no need to futz with sockets, no need to worry about networking layers, just focus on programming against remote resources or on the communication pattern that makes up a protocol.

Unison Distributed is expected to consists of two prime components:

- *a Distributed API*, which includes abstractions useful for programming against multiple distributed nodes, or against data located on remote nodes.
- *a Distributed Runtime*, which is a suite of utilities that run behind the scenes in distributed Unison programs, and which handle bookkeeping for the various abstractions.

This document represents a snapshot of the ongoing process to design Unison's distributed programming facilities, and as such is liable to change.

## APIs

### Remote

Unison's main API for programming against a distributed resource is its `Remote` ability. The basic use case to consider using `Remote` for is for starting parallel tasks at different, remote, nodes in the cluster. 

The main two workhorse functions on `Remote` are `forkAt` and/or `fork`. As its name indicates, `forkAt` allows the user to indicate where they would like the computation to take place, by providing a location `loc` as an argument, while `fork` is indiscriminate about *where* the computation takes place. 

The notion of a *location* is discussed in more detail in the Distributed Runtime section of this document.

The `Remote` ability also includes other useful functions for managing task-parallel computations. Those include `await`, `fail`, `cancel`, amongst others.

Such an API is as follows:

```haskell
unique ability Remote loc task err g where
  forkAt   : loc -> '{g, Remote loc task err g} a -> task a
  fork     : '{g, Remote loc task err g} a -> task a
  location : task a -> loc
  await    : task a -> a
  cancel   : err -> task a -> ()
  fail     : err -> a
```

The notion of an *error* is also discussed in more detail below.

`Remote` can also be thought of as a distributed variant of `Async`. 

In addition to task parallelism, Unison also aims to support straightforward data-centric programming against both stable and ephemeral remote data. 

### Durable

Stable remote data can be represented in Unison using the `Durable` ability. The more straightforward storage API, `Durable`s are what they sound like– remote data that is expected to always be reachable. 

The API for the read and write portions of `Durable`s are as follows:

```haskell
unique ability Durable.W loc d where
  save     : loc -> a -> d a
  location : d a -> loc 

unique ability Durable.R d where
  restore : d a -> a
```

Here, the notion of *location* once again comes up in the API. Importantly, as we will see in subsequent sections of this document, the notion of a location here can be either fine- or coarse-grained. That is, when `save` is called, the `loc` that is used could refer to either a specific compute node, or a logical group of compute nodes (such as a specific instance type in AWS's us-east-1 availability zone.)

What *should* happen regarding garbage collection for `Durable`s is not yet clear. If it is determined that some sort of GCing for `Durable`s should be possible, it might result in changes to this API.

### Ephemeral

Conversely to the notion of a `Durable` is an `Ephemeral`. While a `Durable` is expected to always be reachable, an `Ephemeral` is not. `Ephemeral`s are meant to be used for intermediate remote data that is not important to cache or store long-term. 

Similarly to the API for `Durable`s, the API for the read and write portions of `Ephemeral`s are as follows:

```haskell
unique ability Ephemeral.W d where
  save     : loc -> a -> d a
  location : d a -> loc

unique ability Ephemeral.R d where
  restore : d a -> a
```

Notably here is the notion of `restore` functionality. How could such a function make sense if `Ephemeral`s aren't guaranteed to be accessible at any given moment? One way is through a notion of a *lineage*. Like lineages in Spark, the idea here would be that all `Ephemeral`s must be derived from a `Durable`. This way, keeping track of chains of calls on `Ephemeral`s derived from `Durable`s would make something like a `restore` operation possible– an `Ephemeral` could simply be recomputed based on its backing `Durable`(s) and the cached operations that make up its lineage.

As one might expect, garbage collection for `Ephemeral`s is also an important concern. Unlike for `Durable`s, strategies for GCing remote `Ephemeral`s are more straightforward. For example, the most straightforward way of evicting `Ephemeral`s from memory might be through the use of an LRU cache. 

## Available Models of Computation

### Task Parallelism via Remote

Task parallelism is an old and well-understood parallelization strategy. Unlike other forms of parallelism, task parallelism focuses on tasks (meaningful work) as the main unit to parallelize by "forking" that work and scheduling it on a worker of some kind. This is no different in Unison Distributed using the `Remote` ability. The basic idea is to use `fork` or `forkAt` to spawn a task on some node. Functions like `await` can be used to collect the result performed elsewhere.

As we will see in the section on Examples later on in this document, task parallelism can be used to implement data-parallel interfaces. Contra to task parallelism, data parallelism focuses on data as the main unit to distribute and therefore parallelize. MapReduce-style or batch computing interfaces like RDDs in Spark are examples of data-parallel interfaces. Unison can be used to develop data-parallel libraries using a combination of `Remote`, `Durable`, and `Ephemeral`.

### Message-passing layer

Not all programs that one might wish to implement are task-centric or data-centric. Rather, the implementation of a distributed protocol is communication-centric, requiring explicit messaging and coordination, and task parallelism by way of the Remote ability likely does not suit this implementation use case well.

For that reason, Unison will have a messaging layer that is accessible when needed, but which is generally discouraged. In an effort to remain in a world where communication between independent nodes remains typed, Unison's message-passing layer will be based on the notion of typed channels. These typed channels will be a lower-level API that the `Remote` ability can be handled into.

```haskell
unique ability Channel loc c where
  channel    : loc -> c a
  location   : c a -> loc
 
  -- Send a value to a channel; works anywhere. If the channel
  -- resides on another node, will serialize the `a` and send it
  -- over the network.
  send       : a -> c a -> ()
  
  -- Like `send`, but don't wait for the value to be transferred
  -- before returning.
  sendUnconfirmed : a -> c a -> ()
  
  -- Receive a value from a channel; works anywhere. If the channel
  -- resides on another node, the receive will happen on that node
  -- and the result sent to the caller of `receive`.
  receive    : c a -> a
  
  -- Like `receive` but leave the value in the channel.
  peek       : c a -> a
  
  -- Like `receive`/`peek` but if the channel is currently empty,
  -- returns `None` right away rather than blocking.
  receiveNow : c a -> Optional a
  peekNow    : c a -> Optional a
```

## Error Handling

One aspect that we found ourselves repeatedly returning to was the notion of error handling. When building and debugging a distributed system, useful error handling facilities are essential– having a well-designed notion of what sorts of errors can occur, and being able to explicitly deal with those common error cases can save countless hours of debugging and development time.

This requires us to zoom in on Unison's mechanisms for error handling– which don't yet exist.

Importantly, our philosophy is to keep the notion of an error as generic as possible. For now, we believe the best way to achieve this is by keeping distribution-related errors generic at the level of the `Remote` ability and combinators. This way, APIs like the `Remote` ability and its combinators (e.g., `mapError`, `rescue`, `try`, and others), can leave it to library code built atop of `Remote` to be more opinionated about what exactly an error is, per application.

However, we can use a blanket strategy of allowing client libraries to exclusively decide what an error is. There also exist system-level errors (e.g., Out of Memory Errors) that need to be represented and catchable in the Unison runtime. The represents the need for something core to Unison to be introduced, like a notion of a `SystemError` or the like to be defined as part of Unison's (general, not distributed) runtime. This would enable client libraries to handle system-specific errors however is appropriate for a given application.

*Note that this discussion of error-handling is still very much a work in progress. This discussion touches not just upon Unison Distributed, but Unison's core runtime as well.* 

One possible definition of a generic Unison error type is:

```haskell
unique type Error e
  = User e
  | Timeout Exists
  | Unreachable Exists
  | Cancelled (Error e)
  | System Exists -- e.g., Out of Memory Error

unique type Failure = Failure Link.Type Text Any
unique type TimeoutFailure = -- just a tag

timeout : Text -> Any -> Failure
timeout msg payload = 
  -- `typeLink TimeoutFailure` is similar to classOf[TimeoutFailure] in Scala
  Failure (typeLink TimeoutFailure) msg payload

Remote.fail (timeout "The task timed out!" (param1, param2))
```

Here, we enumerate a few possible generic variants of errors, user-defined errors, errors representing canceled tasks, or system-level errors such as Out of Memory errors. 

Another option for representing errors could be having just a single dynamically-typed error that can be dispatched upon at runtime:

```haskell
unique type Error =
  Error (Optional Link.Type) Exists

-- Example: Make `Remote` concrete in the error type
Remote.fail : Error ->{Remote loc task g} x
```

Given one of these basic notions of an error that could also represent a system-level error, we can go on to define combinators for nicely handling these errors.

```haskell
try : '{Remote loc task err g} a 
   ->{Remote loc task err g} Either err a
try r = ..

mapError : (err -> err2) 
        -> '{Remote loc task err g} a 
        ->{Remote loc task err g} a
mapError f r = match try r with 
  Left e -> Remote.fail (f e)
  Right a -> a

rescue : '{Remote loc task err g}' a 
      -> (err ->{Remote loc task err g} a) 
      ->{Remote loc task err g} a
rescue r onErr = match try r with
  Left err -> onErr err
  Right a  -> a
```

Here, these error-handling combinators provide different ways to do some meaningful work in the place of an error, correct an error, or otherwise handle an error.

As its name suggests `try` is intended to attempt a computation, and wrap the result in an `Either` whether it results in an error or a valid result. `rescue` takes this a step further and in the instance of an error, it attempts to perform some alternate computation and complete the function with the result of that rather than an error.

## Distributed Runtime

Unison's distributed programming facilities cannot be described by APIs and type signatures alone. Rather, there are several behind-the-scenes aspects of a running distributed Unison program that can only be managed by a robust set of background processes that form a distributed runtime.

Aspects such as identity in a running cluster (where exactly is the node identified by some identifier `x`?), scheduling, propagating error information, and management of failure-handling strategies are all runtime affairs.

The Distributed Unison runtime is meant to be a series of running processes on each node in a Unison cluster.

### Locations

As seen earlier in the APIs, Unison is meant to include with it a notion of *location* indicating where a given computation is intended to take place. Locations are meant to be a reference to some notion of compute. This notion can be fine-grained or coarse-grained. That is, location could be a reference referring to a specific node, or it can refer to a group of nodes– e.g., a pool of EC2 nodes spun up in the AWS us-east-1 availability zone.

One idea that we have discussed but not yet worked out is the notion of an *algebra of locations*. This is predicated on the idea that for ephemeral data, one probably does not want to often micromanage the exact location of where a computation takes place. Rather, it might be more desirable to operate on logical groups of ephemerals under weaker constraints, such as "I don't care how the runtime does things, but try to keep these 1000 ephemerals together if you can." We don't know if an algebra of locations would be too rich/flexible, effectively making it overkill, or not. This is an idea worth exploring further.

### Scheduling

How does the Unison Distributed runtime know *which* nodes are most appropriate to schedule tasks on in some pool of nodes?

The Unison Distributed runtime is responsible for understanding the topology of all the locations it has access to. As the Unison Distribute runtime is a running process on all Unison nodes, it keeps track of which node it is running on, and the location of its neighbors. Using pings, it may even be possible to work out which nodes are near (or, quicker to reach) or far (or, slower to reach). 

With such bookkeeping in place, it would then be possible to expose an API for this notion of near vs far (quickness/slowness to reply), and programmers could write libraries that can take advantage of locality. This would effectively give programmers a hook to avoid stragglers for latency-sensitive tasks. And if there is ever to be an algebra of locations, this near vs far (quick vs slow) could even be worked into the algebra of locations.

Ask nodes to run a short computation occasionally to benchmark it in order to determine how overloaded (or not) a node in the system is.

## Examples

### Spark

### Message-Passing (Ping Pong)

### Consensus: Paxos (conceptual)

### Mutual Exclusion

### Gossip something

CRDT

## Open Questions

- GC of Durable and Ephemeral nodes
- Cancellation of lineages
