---
title: 'Raft Implementation with Go: Concurrency and Design Decisions'
date: 2024-02-26
image:
  placement: 2
  caption: 'Source: Original Raft paper'

share: false
profile: true
summary: Diving deep into the concurrency capabilities of Go to efficiently implement Raft consensus algorithm.
---

# Why Go?
After learning Go some time ago for building simple, lightweight, fast mid-level systems, I decided to level up my usage of Go for more complex architectures, especially distributed ones. We hear a lot about why Go is a perfect candidate for implementing distributed systems (not to mention a lot of famous distributed systems in [CNCF landscape](https://landscape.cncf.io/), including Kubernetes, are developed in Go), but I personally couldn't find a simple and clean argument explaining why.

Obviously, after starting this blog post like that, I have to present my own argument, and that's exactly what I am going to do! Let's take a step back for a moment and think about why do we need distributed systems in the first place? You might say it's because monolithic systems cannot scale well, or maintaining such systems are much more flexible since each component (a segment of the entire system) can be maintained by a separate team of developers who publish a complete set of API to other components, thereby enabling different teams for more forward at different development speeds and enhancing isolation. Even recently, with the emergence of edge computing, distributed design of systems seems a necessary step. All of these reasons are correct but are not the **original** reason why software developer started adopting distributed designs. The main incentive behind such design was making a software to be **_fault tolerant_** gainst many form of failures that lurk around, including network partitions, disk failure and bugs at various levels. Having said that, distributed systems also come with many disadvantages, most importantly high complexity and costs. Here my main focus is on high complexity.

The complexity in distributed systems is significantly higher compared to monolithic ones because once our software transitions from a single component to more than one, we have to think about networking aspects as well. A single request and response operation over the network is significantly slower than CPU instruction execution (I recommend taking a look at [magic latency numbers](https://gist.github.com/BillAtHRST/2844543) to get an idea). Even worse, a single communication might not even compete at all because for example an intermediate networking device has dropped packets or due to a [network partition](https://en.wikipedia.org/wiki/Network_partition). A subtle aspect of networking that should not be overlooked is that a designer must contemplate about two sides of networking: connection between system's clients and the system itself, and connection between system's components.


All of these challenges mean if your code should contend with many operations simultaneously. Some of these operations are:
- Check if the internal state is up-to-date
- Wait for the response of a request to a client come back
- Communicate with peer components to complete a procedure
- Do some computations at background


As you can see, each component has to do a lot of things **concurrently** because otherwise it would be extremely inefficient (most often impossible) to perform the required tasks sequentially.

Thus far, I tried to paint a picture of how and why distributed systems are so complex and what is our initial approach towards these challenges. Now let's get back to Go. Go programming language is extremely simple (to a degree that most people often call it boring). It has great support for threading (Go call it _goroutine_). It has a Garbage Collection mechanism that I am so in favor of _in the context of_ distributed systems because it relieves the programmer from thinking about memory allocation/deallocation (don't forget the fact that when working on a distributed system, the developer is thinking about a lot of aspects that don't exist when building a monolithic software). Moreover, it is a lightweight, statically typed, and compiled language, which means software written in Go is fast (we want a component to get ready as fast as possible after a crash, for instance), easy to ship (Docker images must be lightweight for faster deployment and migration), and less error-prone (static types can enable the compiler to assist the programmer). Finally, it has a fine set of synchronization utilities.

Now that I have convinced you to use Go for distributed systems, let's go and discuss some interesting concurrency and design aspects of implementing [Raft consensus algorithm](https://raft.github.io/), which is the cornerstone of projects like [etcd](https://etcd.io/), [CockroachDB](https://www.cockroachlabs.com/), and many others.


{{% callout warning %}}
From now on, I will get into some details of Raft's implementation but I won't cover all of them. Also I will heavily use Raft-specific vocabulary, so if you haven't read [Raft's original paper](https://raft.github.io/raft.pdf), I strongly advise you to do so.
{{% /callout %}}

# Setup and dependencies
Before going into the code and design, I have to describe a common ground for implementation.

{{< figure src="architecture.png" caption="Overall architecture of the system (Source: Original Raft paper)" numbered="true" >}}

According to Figure 1, when we are talking about Raft itself, we are only concerned with the consensus module and the log. We interact with clients and state machines through an interface, but we are not explicitly implementing them. Moreover, we need some predetermined scenarios to run and test our Raft implementation.

After conducting a research, I found [MIT 6.5840's Labs (2023 version)](http://nil.csail.mit.edu/6.5840/2023/) quiet great.

They have provided all of the following:
- A comprehensive infrastructure for developing Raft with having to worry about client and state machine.
- A complete set of testcases to evaluate your implementation.
- Two printing and debugging tools.

I have made some modifications to the provided tools and implemented Raft itself (fully tested). This implementation achieve greater performance (in terms of testcase runtime) compared to the reference runtimes provided at the [Lab's page](http://nil.csail.mit.edu/6.5840/2023/labs/lab-raft.html). My implementation with all test results and updated tools can be found in [this repository](https://github.com/farzad1132/mit-6.5840-lab-implementation).


# Our golden toolkit

In Go's concurrency world, channels and goroutines are not only first-class citizens but also superstars. Here, I just want to some pretty ways of using these two together:

- Implementing [barriers](https://en.wikipedia.org/wiki/Barrier_(computer_science)) using channels: Have your goroutines send/receive something from a controller/parent goroutine.
- Event-driven programming (my favorite): Have your child goroutines wait for a message from a channel. Upon receiving a message, act based on its value.
- Graceful termination: This a special case of event-driven programming, but to emphasize its importance, I mention it separately.
- Distribution of tasks between actors in [Actor programming model](https://en.wikipedia.org/wiki/Actor_model).

To implement all those shining usecases you have to utilize [Time standard library](https://pkg.go.dev/time) and [select](https://go.dev/tour/concurrency/5). A general pattern is as follow:

```go
// General pattern for using select, time and channel

// t is a channel supporting a Timer, Ticker, etc. from time library

// controlCh is sent from the parent goroutine to
// send various events to this goroutine

// You might want to check for termination somewhere
// in this pattern depending our your usecase
for {
  select {
    case msg :<- controlCh:
    switch msg {
      case val1:
        // Perform scenario 1
      case val2:
       // Perform scenario 2
    }
    case <- t:
    // Reset/Initialize something or anything else because
    // for example a timeout has occurred
  }
}
```

{{% callout note %}}
Be aware that `select` can have **default** branch that can be used for non-blocking implementations.
{{% /callout %}}

Before going into the details of the Raft itself, I should mention that Go supports traditional concurrency primitives, such as Mutex, Condition variable, and Wait groups (for more info look at [sync library](https://pkg.go.dev/sync)). However, according to the official documents and many experts, condition variable are not recommend to be used in Go and you can get around with channels and locks instead.

# Implementing Raft

## Implementing state transition capability
In Raft, we have three states and their relation and transition conditions are as follow:

{{< figure src="transition.png" caption="State transition in Raft (Source: Original Raft paper)" numbered="true" >}}

This might look simple to implement in a client and server, but in a **Raft instance, which is a client and server simultaneously**, it's not that easy, and from my experience, a lot of people who are new to concurrency have difficulty implementing something like that.

To figure out what we have to do, let's break out the peaces:

- We have several RPC handlers that can respond to `RequestVote`, `appendEntries`, and `InstalSnapshot` RPCs.
- We cannot loop inside a loop to check for state transition events because this would hurt the performance significantly (a lot of sleep and unnecessary operations)
- Trigger for state transition can come from various goroutines
- If state transition happens, we cannot stop all previous operations and start new ones since some functionalities, such as handling RPC requests, overlap in different states.

If you remember our golden toolkit that I introduced above, we can solve this issue with event-driven programming:

```go
func (rf *Raft) main() {
  debug.Debug(debug.DInfo, rf.me, "Starting main.")
  for !rf.killed() {

    // Wait for state transition
    state := <-rf.controlCh

    // Do not block inside this switch statement
    switch state {
    case Candidate:
      debug.Debug(debug.DInfo, rf.me, "Starting an election...")
      go startElection(rf)
    case Leader:
      debug.Debug(debug.DInfo, rf.me, "Starting leader routine.")
      go startLeader(rf)
    }
  }
}
```

Above function (using a simplified version of general concurrency pattern) runs in a separate goroutine from the start. As you can see, it waits for new events (states in this case) from control cannel, which is accessible in other parts of the program. We don't do anything for `Follower` state since it doesn't require any beyond responding to RPCs and checking election timeout.

{{% callout note %}}
`main` function (running as a goroutine) just kick starts other functionalities as separate goroutines and it doesn't block at all. This is necessary to keep you Raft instance responsive for new transitions.
{{% /callout %}}

## Autonomy of child goroutines

In the previous subsection, `main` function doesn't get any termination event from `rf.controlCh`. You might think coordinating every goroutine tightly is more efficient. However, doing such thing increases the complexity of your implementation substantially. Instead, it's easier and faster to implement conditions for termination internally inside your child goroutines and not worry about them anymore when you are working on your parent (top level) goroutines.

For example, look at the `instanceWatchDog` goroutines, which is responsible for sending heartbeats to `Follower` instances from the `Leader`.

```go
func instanceWatchDog(server int, rf *Raft, term int) {
  debug.Debug(debug.DInfo, rf.me, "Starting watchdog for %v.", server)

  // ... some initialization

  for {
    rf.mu.Lock()
    // Consistency check
    if rf.killed() || rf.state != Leader || rf.currentTerm != term {
      debug.Debug(debug.DConsist, rf.me, "Watchdog state is inconsistent. isKilled:%v, state:%v, curTerm:%v, watchdogTerm: %v",
        rf.killed(), rf.state, rf.currentTerm, term)
      rf.mu.Unlock()
      return
    }
    
    rf.mu.Unlock()

    // ... other functionalities implemented using general pattern

  }
}
```

As you can see, this function constantly performs consistency checks. If these checks fail, it self terminates itself. In my opinion the most important condition is checking for `current term`. In my design, **every watchdog instance is dedicated to a specific `term`**. As a result, in the next `term`, the leader doesn't have to worry about old watchdogs and it can start new ones safely without compromising performance. **_This is exactly one of the usecases in distributed systems that having a programming language with garbage collection mechanism make a lot of difference because we, as programmers, don't have worry about garbage from old watchdogs and the code will be much simpler and cleaner._**

## Simultaneously handling client-like and server-like responsibilities

As I said before, in Raft, specially when you are in `Candidate` and `Leader` states, you have to engage in operations that are client-like (meaning that you have to start it and wait for its result) and server-like (meaning that you have wait for some other entity to initiate a procedure to produce the results). Again, from my experience, a lot of new developers have difficulty handling these scenarios. In these cases you just have to remember a simple sentence: **_In distributed systems, all you need are watchdogs and heartbeats_**.

{{< figure src="everywhere.jpg" numbered="true" >}}


Let's discuss them more closely. **watchdog**s are entities that wait for some event, such as a timeout, to perform something. In the case of Raft, the `Leader` with create a watchdog for every `Follower`. This watchdog is responsible for sending `AppendEntries` RPC. The following is a part of this watchdog implementation:

```go
select {
  case <-timer.C:
    debug.Debug(debug.DTimer, rf.me, "Watchdog for %v timeout.", server)
    timer.Reset(timeout)
    go appendEntriesWrapper(rf, server)
  case flag := <-ch:
    switch flag {
    case 0:
      // Received reply of AppendEntries RPC with zero entries
      
      // do something like timer.Reset(timeout)
    case 1:
      // Received reply of AppendEntries RPC with non-zero entries

      // Do something like this:
      // timer.Reset(timeout)
      // go appendEntriesWrapper(rf, server)
    case 2:
      // Received a new entry from the client
      timer.Reset(timeout)
      go appendEntriesWrapper(rf, server)
    }
    }
```

You might ask why am I not sending `AppendEntries` Response to facilitate committing new entries in `Follower`s. This this is because on of the testcases provided by course instructors named `TestCount2B` limits the number of RPCs allowed for the implementation. Consequently, I am sending this RPC only when receiving a new entry or after a timeout in batch.


Now, let's talk about **heartbeats**. These are simple RPC to just check whether a specific entity is alive and responsive or not. Usually, they are used in conjunction with timers to make sure something gets done in a specific amount of time. For instance, in our watchdog implementation, one of the `select` branches is waiting on a timer. When this timer goes off, we reset it and send a heartbeat.

# Final notes

- Do not block on any interface to upper-level services, such as the state machine, that are using Raft for distributed consensus. This is important because you don't know how these upper-level services might use this interface.
- Do not forget about careful logging. Otherwise, debugging such complex distributed systems would be almost impossible. More about logging in the [Lab's page](http://nil.csail.mit.edu/6.5840/2023/labs/lab-raft.html) [my solutions repository](https://github.com/farzad1132/mit-6.5840-lab-implementation).


If you had any questions or notes regarding the implementation of this post, please don't hesitate to let me know via email or any other contact point.


