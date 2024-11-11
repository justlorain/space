---
title: "A Deep Dive into Raft Log of etcd raft"
description: "Understanding etcd's Raft Implementation: A Deep Dive into Raft Log"
publishDate: "11 Nov 2024"
tags: ["tutorial", "go", "distributedsystem", "database"]
---

## Introduction

This article will introduce and analyze the design and implementation of the Raft Log module in etcd's Raft, starting from the log in the Raft consensus algorithm. The goal is to help readers better understand the implementation of etcd's Raft and provide a possible approach for implementing similar scenarios.

## Raft Log Overview

The Raft consensus algorithm is essentially a **replicated state machine**, with the goal of replicating a series of logs in the same way across a server cluster. These logs enable the servers in the cluster to reach a consistent state.

In this context, the logs refer to the Raft Log. Each node in the cluster has its own Raft Log, which is composed of a series of log entries. A log entry typically contains three fields:

- **Index:** The index of the log entry
- **Term:** The leader's term when the log entry was created
- **Data:** The data contained in the log entry, which could be specific commands, etc.

It's important to note that the index of the Raft Log starts at **1**, and only the leader node can create and replicate the Raft Log to follower nodes.

When a log entry is persistently stored on the majority of nodes in the cluster (e.g., 2/3, 3/5, 4/7), it is considered **committed**.

When a log entry is applied to the state machine, it is considered **applied**.

![1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zzox0v9evpg278wxxg6g.png)

## etcd's raft implementation overview

etcd raft is a Raft algorithm library written in Go, widely used in systems like etcd, Kubernetes, CockroachDB, and others.

The primary characteristic of etcd raft is that it only implements the core part of the Raft algorithm. Users must implement network transmission, disk storage, and other components involved in the Raft process themselves (although etcd provides default implementations).

Interacting with the etcd raft library is somewhat straightforward: it tells you which data needs to be persisted and which messages need to be sent to other nodes. Your responsibility is to handle the storage and network transmission processes and inform it accordingly. It doesn’t concern itself with the details of how you implement these operations; it simply processes the data you submit and, based on the Raft algorithm, tells you the next steps.

In the implementation of etcd raft’s code, this interaction model is seamlessly combined with Go's unique channel feature, making the etcd raft library truly distinctive.

## How to Implement Raft Log

### log and log_unstable

In etcd raft, the main implementation of Raft Log is located in the `log.go` and `log_unstable.go` files, with the primary structures being `raftLog` and `unstable`. The `unstable` structure is also a field within `raftLog`.

- **raftLog** is responsible for the main logic of the Raft Log. It can access the node’s log storage state through the `Storage` interface provided to the user.
- **unstable**, as its name suggests, contains log entries that haven’t been persisted yet, meaning uncommitted logs.

etcd raft manages the logs within the algorithm by coordinating `raftLog` and `unstable`.

### Core Fields of raftLog and unstable

To simplify the discussion, this article will focus only on the processing logic of log entries, without addressing snapshot handling in etcd raft.

```go
type raftLog struct {
	storage Storage
	unstable unstable
	committed uint64
	applying uint64
	applied uint64
}
```

Core fields of `raftLog`:

- **storage:** A storage interface implemented by the user, used to retrieve log entries that have already been persisted.
- **unstable:** Stores unpersisted logs. For example, when the Leader receives a request from a client, it creates a log entry with its Term and appends it to the unstable logs.
- **committed:** Known as `commitIndex` in the Raft paper, it represents the index of the last known committed log entry.
- **applying:** The highest index of a log entry that is currently being applied.
- **applied:** Known as `lastApplied` in the Raft paper, it is the highest index of a log entry that has been applied to the state machine.

```go
type unstable struct {
	entries []pb.Entry
	offset uint64
	offsetInProgress uint64
}
```

Core fields of `unstable`:

- **entries:** The unpersisted log entries, stored in memory as a slice.
- **offset:** Used to map log entries in `entries` to the Raft Log, where `entries[i] = Raft Log[i+offset]`.
- **offsetInProgress:** Indicates entries that are currently being persisted. The entries in progress are represented by `entries[:offsetInProgress-offset]`.

The core fields in `raftLog` are straightforward and can easily be related to the implementation in the Raft paper. However, the fields in `unstable` might seem more abstract. The following example aims to help clarify these concepts.

Assume we already have 5 log entries persisted in our Raft Log. Now, we have 3 log entries stored in `unstable`, and these 3 log entries are currently being persisted. The situation is as shown below:

![2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/69rn0ydw4qhd0blbsrdn.png)

`offset=6` indicates that the log entries at positions 0, 1, and 2 in `unstable.entries` correspond to positions 6 (0+6), 7 (1+6), and 8 (2+6) in the actual Raft Log. With `offsetInProgress=9`, we know that `unstable.entries[:9-6]`, which includes the three log entries at positions 0, 1, and 2, are all being persisted.

> The reason `offset` and `offsetInProgress` are used in `unstable` is that `unstable` does not store all Raft Log entries.

### When to Interact

Since we are focusing only on the Raft Log processing logic, "when to interact" here refers to when etcd raft will pass the log entries that need to be persisted by the user.

#### User Side

etcd raft interacts with the user primarily through methods in the `Node` interface. The `Ready` method returns a channel that allows the user to receive data or instructions from etcd raft.

```go
type Node interface {
    ...
    Ready() <-chan Ready
    ...
}
```

The `Ready` struct received from this channel contains log entries that need processing, messages that should be sent to other nodes, the current state of the node, and more.

For our discussion on Raft Log, we only need to focus on the `Entries` and `CommittedEntries` fields:

- **Entries:** Log entries that need to be persisted. Once these entries are persisted, they can be retrieved using the `Storage` interface.
- **CommittedEntries:** Log entries that need to be applied to the state machine.

```go
type Ready struct {
	*SoftState
	pb.HardState
	ReadStates []ReadState
	Entries []pb.Entry // persist it
	Snapshot pb.Snapshot
	CommittedEntries []pb.Entry // commit it
	Messages []pb.Message
	MustSync bool
}
```

After processing the logs, messages, and other data passed through `Ready`, we can call the `Advance` method in the `Node` interface to inform etcd raft that we have completed its instructions, allowing it to receive and process the next `Ready`.

> etcd raft offers an `AsyncStorageWrites` option, which can enhance node performance to some extent. However, we are not considering this option here.

#### etcd raft Side

On the user side, the focus is on handling the data in the received `Ready` struct. On the etcd raft side, the focus is on determining when to pass a `Ready` struct to the user and what actions to take afterward.

I have summarized the main methods involved in this process in the following diagram, which shows the general sequence of method calls (note that this only represents the approximate order of calls):

![3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ybcytyr844yazqk6wvh7.png)

You can see that the entire process is a loop. Here, we'll outline the general function of these methods, and in the subsequent write-flow analysis, we’ll delve into how these methods operate on the core fields of `raftLog` and `unstable`.

- **HasReady:** As the name suggests, it checks whether there is a `Ready` struct that needs to be passed to the user. For example, if there are unpersisted log entries in `unstable` that are not currently in the process of being persisted, `HasReady` will return `true`.
- **readyWithoutAccept:** Called after `HasReady` returns `true`, this method creates the `Ready` struct to be returned to the user, including the log entries that need to be persisted and those marked as committed.
- **acceptReady:** Called after etcd raft passes the `Ready` struct created by `readyWithoutAccept` to the user. It marks the log entries returned in `Ready` as in the process of being persisted and applied, and creates a “callback” to be invoked when the user calls `Node.Advance`, marking the log entries as persisted and applied.
- **Advance:** Executes the “callback” created in `acceptReady` after the user calls `Node.Advance`.

### How to Define Committed and Applied

There are two important points to consider here:

**1. Persisted ≠ Committed**

As defined initially, a log entry is considered committed only when it has been persisted by the majority of nodes in the Raft cluster. So even if we persist the `Entries` returned by etcd raft through `Ready`, these entries cannot yet be marked as committed.

However, when we call the `Advance` method to inform etcd raft that we have completed the persistence step, etcd raft will evaluate the persistence status across other nodes in the cluster and mark some log entries as committed. These entries are then provided to us through the `CommittedEntries` field of the `Ready` struct so we can apply them to the state machine.

Thus, when using etcd raft, the timing for marking entries as committed is managed internally, and users only need to fulfill the persistence prerequisites.

> Internally, commitment is achieved by calling the `raftLog.commitTo` method, which updates `raftLog.committed`, corresponding to the `commitIndex` in the Raft paper.

**2. Committed ≠ Applied**

After the `raftLog.commitTo` method is called within etcd raft, the log entries up to the `raft.committed` index are considered committed. However, entries with indices in the range `lastApplied < index <= committedIndex` have not yet been applied to the state machine. etcd raft will return these committed but un-applied entries in the `CommittedEntries` field of `Ready`, allowing us to apply them to the state machine. Once we call `Advance`, etcd raft will mark these entries as applied.

The timing for marking entries as applied is also handled internally in etcd raft; users only need to apply the committed entries from `Ready` to the state machine.

> Another subtle point is that, in Raft, only the Leader can commit entries, but all nodes can apply them.

## Processing Flow of a Write Request

Here, we’ll connect all the previously discussed concepts by analyzing the flow of etcd raft as it handles a write request.

### Initial State

To discuss a more general scenario, we’ll start with a **Raft Log that has already committed and applied three log entries**.

![4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2c03r5nx53fj1yu5zt6i.png)

In the illustration, **green** represents `raftLog` fields and the stored log entries in `Storage`, while **red** represents `unstable` fields and the unpersisted log entries stored in `entries`.

Since we have committed and applied three log entries, both `committed` and `applied` are set to 3. The `applying` field holds the index of the highest log entry from the previous application, which is also 3 in this case.

At this point, no requests have been initiated, so `unstable.entries` is empty. The next log index in the Raft Log is 4, making `offset` 4. Since no logs are currently being persisted, `offsetInProgress` is also set to 4.

### Issue a Request

Now, we initiate a request to append two log entries to the Raft Log.

![5](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/l56ddmnd15jkb5fpndfa.png)

As shown in the illustration, the appended log entries are stored in `unstable.entries`. At this stage, no changes are made to the index values recorded in the core fields.

### HasReady

Remember the `HasReady` method? `HasReady` checks if there are unpersisted log entries and, if so, returns `true`.

The logic for determining the presence of unpersisted log entries is based on whether the length of `unstable.entries[offsetInProgress-offset:]` is greater than 0. Clearly, in our case:

```go
len(unstable.entries[4 - 4:]) == 2
```

indicating that there are two unpersisted log entries, so `HasReady` returns `true`.

![6](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/73ubbq5y0de5jtteyjqd.png)

### readyWithoutAccept

The purpose of `readyWithoutAccept` is to create the `Ready` struct to be returned to the user. Since we have two unpersisted log entries, `readyWithoutAccept` will include these two log entries in the `Entries` field of the returned `Ready`.

![7](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p941r8uxcvwzsbrcqiv9.png)

### acceptReady

`acceptReady` is called after the `Ready` struct is passed to the user.

![8](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t5n89wxleo8dyvl1ulrz.png)

`acceptReady` updates the index of log entries that are in the process of being persisted to 6, meaning that log entries within the range `[4, 6)` are now marked as being persisted.

### Advance

After the user persists the `Entries` in `Ready`, they call `Node.Advance` to notify etcd raft. Then, etcd raft can execute the "callback" created in `acceptReady`.

![9](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kpfz45r1uspbyss7vhlk.png)

This "callback" clears the already persisted log entries in `unstable.entries`, then sets `offset` to `Storage.LastIndex + 1`, which is 6.

### Commit Log Entries

We assume that these two log entries have already been persisted by the majority of nodes in the Raft cluster, so we can mark these two log entries as committed.

![10](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7v22rj7he4lnn1cblcdf.png)

### HasReady

Continuing with our loop, `HasReady` detects the presence of log entries that are committed but not yet applied, so it returns `true`.

![11](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gon2f2ujc9bqw7wt1l70.png)

### readyWithoutAccept

`readyWithoutAccept` returns a `Ready` containing log entries (4, 5) that are committed but have not been applied to the state machine.

These entries are calculated as `low, high := applying+1, committed+1`, in a left-open, right-closed interval.

![12](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1nnl2dlaahw9au37jbjw.png)

### acceptReady

`acceptReady` then marks the log entries `[4, 5]` returned in `Ready` as being applied to the state machine.

![13](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5bqsa1wv2gjwenk4xqfg.png)

### Advance

After the user calls `Node.Advance`, etcd raft executes the "callback" and updates `applied` to 5, indicating that the log entries at index 5 and earlier have all been applied to the state machine.

![14](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9wdwpfkiqd2nv1dgswy6.png)

### Final State

This completes the processing flow for a write request. The final state is as shown below, which can be compared to the initial state.

![15](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9mh8cv82f4hyll8nn2dy.png)

## Summary

We started with an overview of the Raft Log, gaining an understanding of its basic concepts, followed by an initial look at the etcd raft implementation. We then delved deeper into the core modules of Raft Log within etcd raft and considered important questions. Finally, we tied everything together through a complete analysis of a write request flow.

I hope this approach helps you gain a clear understanding of the etcd raft implementation and develop your own insights into the Raft Log.

That concludes this article. If there are any mistakes or questions, feel free to reach out via private message or leave a comment.

BTW, [raft-foiver](https://github.com/B1NARY-GR0UP/raft) is a simplified version of etcd raft that I implemented, retaining all the core logic of Raft and optimized according to the process in the Raft paper. I’ll release a separate post introducing this library in the future. If you're interested, feel free to Star, Fork, or PR!

## Reference

- https://github.com/B1NARY-GR0UP/raft
- https://github.com/etcd-io/raft