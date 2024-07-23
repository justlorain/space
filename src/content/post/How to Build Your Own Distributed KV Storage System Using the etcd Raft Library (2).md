---
title: "Build KV Storage System Using the etcd Raft Library (2)"
description: "How to Build Your Own Distributed KV Storage System Using the etcd Raft Library"
publishDate: "23 Jul 2024"
tags: ["webdev", "go", "tutorial", "distributedsystem"]
---

## Introduction

In the [first article](https://www.justlorain.space/blog/how-to-build-your-own-distributed-kv-storage-system-using-the-etcd-raft-library/), we learned and became familiar with the structure of raftexample and the processing flow of a write request. In this article, we will interpret the log compaction and snapshot handling logic in raftexample.

## Log Compaction and Snapshot

The Raft log continuously grows as the cluster operates normally and processes client requests. In practical applications, we need a mechanism to limit the unlimited growth of the log to avoid causing availability issues. Raft uses the snapshot mechanism to compact the log, as shown in the figure below:

![snapshot example](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z9paxbxg9px2tduwr6cq.png)

After creating a snapshot, all **committed entries** in the Raft log are included in a snapshot. This snapshot compresses and merges the state of the state machine for the last committed entry and all previous entries and saves the Index and Term of the last entry included in the snapshot for consistency checks during the AppendEntries RPC (log synchronization). To support cluster membership changes, the snapshot should also save the latest cluster configuration (for simplicity, this is not shown in the figure).

The timing of snapshot creation is also a topic worth discussing. A simple strategy is to create a snapshot when the log reaches a configured size threshold (in bytes).

## Creating Snapshots

**1. In raftexample, the strategy for creating snapshots is to do so when the number of log entries reaches a specified threshold. This strategy is executed by the `maybeTriggerSnapshot` method, which is called every time data is received from the raft library (Ready).**

This method first calculates the number of committed log entries since the last snapshot. If this number has not reached the specified threshold (`snapCount`), it returns immediately without creating a snapshot.

```go
if rc.appliedIndex - rc.snapshotIndex <= rc.snapCount {
    return
}
```

Next, it blocks and waits until all the committed log entries received from the raft library that need to be applied (`rc.entriesToApply(rd.CommittedEntries)`) are applied to the state machine (kvstore) or the server shuts down.

```go
// wait until all committed entries are applied (or server is closed)
if applyDoneC != nil {
    select {
    case <-applyDoneC:
    case <-rc.stopc:
        return
    }
}
```

Then the actual snapshot creation logic follows:

- **getSnapshot**: Retrieves the JSON serialized data of all key-value pairs stored in the current kvstore state machine;

- **CreateSnapshot**: Creates a snapshot. `CreateSnapshot` calculates the last included index and last included term based on the committed log index (`appliedIndex`). The snapshot also includes the latest cluster configuration information (`confState`) and the state of the state machine (`data`).

  This strictly follows the requirements mentioned in the paper that a snapshot must include all necessary information.

- **saveSnap**: Writes the created snapshot to disk and records the snapshot metadata in the WAL to enable state recovery from the WAL.

```go
log.Printf("start snapshot [applied index: %d | last snapshot index: %d]", rc.appliedIndex, rc.snapshotIndex)
data, err := rc.getSnapshot()
if err != nil {
    log.Panic(err)
}
snap, err := rc.raftStorage.CreateSnapshot(rc.appliedIndex, &rc.confState, data)
if err != nil {
    panic(err)
}
if err := rc.saveSnap(snap); err != nil {
    panic(err)
}
```

In the implementation of MemoryStorage, since `CreateSnapshot` does not actually compress the Raft Log, we need to manually call the `Compact` method to delete all entries before `compactIndex`. Otherwise, creating snapshots without deleting the Raft Log would render the snapshot creation meaningless. Here, we retain `snapshotCatchUpEntriesN` log entries while deleting the entries to allow some slow followers to catch up with the leader.

```go
// keep some in memory log entries for slow followers.
compactIndex := uint64(1)
// snapshotCatchUpEntriesN represents the number of log entries retained after a snapshot is triggered.
if rc.appliedIndex > snapshotCatchUpEntriesN {
    compactIndex = rc.appliedIndex - snapshotCatchUpEntriesN
}
if err := rc.raftStorage.Compact(compactIndex); err != nil {
    if err != raft.ErrCompacted {
        panic(err)
    }
} else {
    log.Printf("compacted log at index %d", compactIndex)
}
```

Finally, update the snapshot progress to the current last committed log entry.

```go
rc.snapshotIndex = rc.appliedIndex
```

**2. In addition to nodes creating snapshots periodically based on their snapshot creation strategy, followers may also receive snapshots from the leader to help with log synchronization in cases such as new nodes joining the cluster or log entries lagging too far behind.**

When the Snapshot field of Ready is not empty, persist the snapshot, then apply the snapshot (`ApplySnapshot`), and notify the kvstore module to load the snapshot by sending a `nil` signal to `commitC`.

```go
if !raft.IsEmptySnap(rd.Snapshot) {
    rc.saveSnap(rd.Snapshot)
}
// rd.HardState + rd.Entries = persistent state on all servers
rc.wal.Save(rd.HardState, rd.Entries)
if !raft.IsEmptySnap(rd.Snapshot) {
    rc.raftStorage.ApplySnapshot(rd.Snapshot)
    // Send a signal to load the snapshot. 
    // The kvstore will restore the state machine from the snapshot.
    rc.publishSnapshot(rd.Snapshot)
}
```

Upon receiving the `nil` signal, kvstore loads the previously saved snapshot from the disk through `saveSnap`, and `recoverFromSnapshot` deserializes the JSON data of all key-value pairs saved in the snapshot to overwrite the state machine's (`map[string]string`) storage.

```go
for commit := range commitC {
    // signaled by raftNode.publicSnapshot rc.commitC <- nil
    if commit == nil {
        snapshot, err := s.loadSnapshot()
        if err != nil {
            log.Panic(err)
        }
        if snapshot != nil {
            log.Printf("loading snapshot at term %d and index %d", snapshot.Metadata.Term, snapshot.Metadata.Index)
            if err := s.recoverFromSnapshot(snapshot.Data); err != nil {
                log.Panic(err)
            }
        }
        continue
    }
    ...
}
```

## Recovering from Snapshots

Through the `maybeTriggerSnapshot` method, raftexample creates a snapshot and persists it to disk when the log entries reach a set threshold. Next, let's look at how raftexample uses these snapshots on disk for recovery.

**1. During the startup of the raft module, the Raft Log is reconstructed using the `replayWAL` method.**

- The `loadSnapshot` method loads the previously saved snapshot from disk using `saveSnap`.
- `openWAL` reconstructs the WAL based on the loaded snapshot's Index and Term.
- `ApplySnapshot`, `SetHardState`, and `Append` are used to restore MemoryStorage, which is the Raft Log (MemoryStorage is in-memory, so it needs the WAL for recovery).

> The HardState and Entries saved in the WAL were persisted after receiving data in the Ready state.
>
> `rc.wal.Save(rd.HardState, rd.Entries)`

```go
func (rc *raftNode) replayWAL() *wal.WAL {
    log.Printf("replaying WAL of member %d", rc.id)
    snapshot := rc.loadSnapshot()
    w := rc.openWAL(snapshot)
    _, st, ents, err := w.ReadAll()
    if err != nil {
        log.Fatalf("raftexample: failed to read WAL (%v)", err)
    }
    rc.raftStorage = raft.NewMemoryStorage()
    if snapshot != nil {
        rc.raftStorage.ApplySnapshot(*snapshot)
    }
    rc.raftStorage.SetHardState(st)

    // append to storage so raft starts at the right place in log
    rc.raftStorage.Append(ents)

    return w
}
```

After starting the Raft module, in the `serveChannels` method of raftexample, which handles interactions with the raft library, we can use the `Snapshot` method to retrieve the snapshot applied to MemoryStorage in `replayWAL` and reconstruct the current node's cluster configuration (`confState`), snapshot index (`snapshotIndex`), and committed log index (`appliedIndex`).

```go
snap, err := rc.raftStorage.Snapshot()
if err != nil {
    panic(err)
}

rc.confState = snap.Metadata.ConfState
rc.snapshotIndex = snap.Metadata.Index
rc.appliedIndex = snap.Metadata.Index
```

**2. Besides the raft module reconstructing the Raft Log from snapshots, the kvstore module also recovers the state machine using snapshots.**

Similarly, it uses `loadSnapshot` to load the snapshot and then `recoverSnapshot` to deserialize the JSON data to recover the state machine.

```go
func newKVStore(snapshotter *snap.Snapshotter, proposeC chan<- string, commitC <-chan *commit, errorC <-chan error) *kvstore {
    s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
    snapshot, err := s.loadSnapshot()
    if err != nil {
        log.Panic(err)
    }
    if snapshot != nil {
        log.Printf("loading snapshot at term %d and index %d", snapshot.Metadata.Term, snapshot.Metadata.Index)
        if err := s.recoverFromSnapshot(snapshot.Data); err != nil {
            log.Panic(err)
        }
    }

    go s.readCommits(commitC, errorC)
    return s
}
```

## Summary

That's all for this article. We have analyzed the log compaction and snapshot handling logic in raftexample from two perspectives: **creating snapshots** and **recovering from snapshots**. I hope this helps you better understand how to use the etcd raft library to build your own distributed KV storage service.

If there are any mistakes or issues, please feel free to comment or message me directly. Thank you.

## References

- https://github.com/etcd-io/etcd/tree/main/contrib/raftexample
- https://github.com/etcd-io/raft
- https://raft.github.io/raft.pdf
