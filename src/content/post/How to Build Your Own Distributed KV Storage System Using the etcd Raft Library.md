---
title: "Build KV Storage System Using the etcd Raft Library"
description: "How to Build Your Own Distributed KV Storage System Using the etcd Raft Library"
publishDate: "5 Jul 2024"
tags: ["webdev", "go", "tutorial", "distributedsystem"]
---

## Introduction

[raftexample](https://github.com/etcd-io/etcd/tree/main/contrib/raftexample) is an example provided by etcd that demonstrates the use of the etcd raft consensus algorithm library. raftexample ultimately implements a distributed key-value storage service that provides a REST API.

This article will read and analyze the code of raftexample, hoping to help readers better understand how to use the etcd raft library and the implementation logic of the raft library.

## Architecture

The architecture of raftexample is very simple, with the main files as follows:

- **main.go:** Responsible for organizing the interaction between the raft module, the httpapi module, and the kvstore module;
- **raft.go:** Responsible for interacting with the raft library, including submitting proposals, receiving RPC messages that need to be sent, and performing network transmission, etc.;
- **httpapi.go:** Responsible for providing the REST API, serving as the entry point for user requests;
- **kvstore.go:** Responsible for persistently storing committed log entries, equivalent to the state machine in the raft protocol.

## The Processing Flow of a Write Request

A write request arrives in the `ServeHTTP` method of the httpapi module via an HTTP PUT request.

```shell
curl -L http://127.0.0.1:12380/key -XPUT -d value
```

After matching the HTTP request method via `switch`, it enters the PUT method processing flow:

- Read the content from the HTTP request body (i.e., the value);
- Construct a proposal through the `Propose` method of the kvstore module (adding a key-value pair with key as key and value as value);
- Since there is no data to return, respond to the client with 204 StatusNoContent;

> **The proposal is submitted to the raft algorithm library through the `Propose` method provided by the raft algorithm library.**
>
> **The content of a proposal can be adding a new key-value pair, updating an existing key-value pair, etc.**

```go
// httpapi.go
v, err := io.ReadAll(r.Body)
if err != nil {
    log.Printf("Failed to read on PUT (%v)\n", err)
    http.Error(w, "Failed on PUT", http.StatusBadRequest)
    return
}
h.store.Propose(key, string(v))
w.WriteHeader(http.StatusNoContent)
```

Next, let's look into the `Propose` method of the kvstore module to see how a proposal is constructed and processed.

In the `Propose` method, we first encode the key-value pair to be written using gob, and then pass the encoded content to `proposeC`, a channel responsible for transmitting proposals constructed by the kvstore module to the raft module.

```go
// kvstore.go
func (s *kvstore) Propose(k string, v string) {
	var buf strings.Builder
	if err := gob.NewEncoder(&buf).Encode(kv{k, v}); err != nil {
		log.Fatal(err)
	}
	s.proposeC <- buf.String()
}
```

The proposal constructed by kvstore and passed to `proposeC` is received and processed by the `serveChannels` method in the raft module.

After confirming that `proposeC` has not been closed, the raft module submits the proposal to the raft algorithm library for processing using the `Propose` method provided by the raft algorithm library.

```go
// raft.go
select {
    case prop, ok := <-rc.proposeC:
    if !ok {
        rc.proposeC = nil
    } else {
        rc.node.Propose(context.TODO(), []byte(prop))
    }
```

After a proposal is submitted, it follows the raft algorithm process. The proposal will eventually be forwarded to the leader node (if the current node is not the leader and you allow followers to forward proposals, controlled by the `DisableProposalForwarding` configuration). The leader will add the proposal as a log entry to its raft log and synchronize it with other follower nodes. After being deemed committed, it will be applied to the state machine and the result will be returned to the user.

However, since the etcd raft library itself does not handle communication between nodes, appending to the raft log, applying to the state machine, etc., the raft library only prepares the data required for these operations. The actual operations must be performed by us.

Therefore, we need to receive this data from the raft library and process it accordingly based on its type. The `Ready` method returns a read-only channel through which we can receive the data that needs to be processed.

> **It should be noted that the received data includes multiple fields, such as snapshots to be applied, log entries to be appended to the raft log, messages to be transmitted over the network, etc.**

Continuing with our write request example (leader node), after receiving the corresponding data, we need to persistently save snapshots, `HardState`, and `Entries` to handle issues caused by server crashes (e.g., a follower voting for multiple candidates). `HardState` and `Entries` together comprise the `Persistent state on all servers` as mentioned in the paper. After persistently saving them, we can apply the snapshot and append to the raft log.

Since we are currently the leader node, the raft library will return `MsgApp` type messages to us (corresponding to `AppendEntries` RPC in the paper). We need to send these messages to the follower nodes. Here, we use the rafthttp provided by etcd for node communication and send the messages to follower nodes using the `Send` method.

```go
// raft.go
case rd := <-rc.node.Ready():
    if !raft.IsEmptySnap(rd.Snapshot) {
        rc.saveSnap(rd.Snapshot)
    }
    rc.wal.Save(rd.HardState, rd.Entries)
    if !raft.IsEmptySnap(rd.Snapshot) {
        rc.raftStorage.ApplySnapshot(rd.Snapshot)
        rc.publishSnapshot(rd.Snapshot)
    }
    rc.raftStorage.Append(rd.Entries)
    rc.transport.Send(rc.processMessages(rd.Messages))
    applyDoneC, ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries))
    if !ok {
        rc.stop()
        return
    }
    rc.maybeTriggerSnapshot(applyDoneC)
    rc.node.Advance()
```

Next, we use the `publishEntries` method to apply the committed raft log entries to the state machine. As mentioned earlier, in raftexample, the kvstore module acts as the state machine. In the `publishEntries` method, we pass the log entries that need to be applied to the state machine to `commitC`. Similar to the earlier `proposeC`, `commitC` is responsible for transmitting the log entries that the raft module has deemed committed to the kvstore module for application to the state machine.

```go
// raft.go
rc.commitC <- &commit{data, applyDoneC}
```

In the `readCommits` method of the kvstore module, messages read from `commitC` are gob-decoded to retrieve the original key-value pairs, which are then stored in a map structure within the kvstore module.

```go
// kvstore.go
for commit := range commitC {
	...
    for _, data := range commit.data {
        var dataKv kv
        dec := gob.NewDecoder(bytes.NewBufferString(data))
        if err := dec.Decode(&dataKv); err != nil {
            log.Fatalf("raftexample: could not decode message (%v)", err)
        }
        s.mu.Lock()
        s.kvStore[dataKv.Key] = dataKv.Val
        s.mu.Unlock()
    }
    close(commit.applyDoneC)
}
```

Returning to the raft module, we use the `Advance` method to notify the raft library that we have finished processing the data read from the `Ready` channel and are ready to process the next batch of data.

Earlier, on the leader node, we sent `MsgApp` type messages to the follower nodes using the `Send` method. The follower node's rafthttp listens on the corresponding port to receive requests and return responses. Whether it's a request received by a follower node or a response received by a leader node, it will be submitted to the raft library for processing through the `Step` method.

> **`raftNode` implements the `Raft` interface in rafthttp, and the `Process` method of the `Raft` interface is called to handle the received request content (such as `MsgApp` messages).**

```go
// raft.go
func (rc *raftNode) Process(ctx context.Context, m raftpb.Message) error {
	return rc.node.Step(ctx, m)
}
```

The above describes the complete processing flow of a write request in raftexample.

## Summary

This concludes the content of this article. By outlining the structure of raftexample and detailing the processing flow of a write request, I hope to help you better understand how to use the etcd raft library to build your own distributed KV storage service.

If there are any mistakes or issues, please feel free to comment or message me directly. Thank you.

## References

- https://github.com/etcd-io/etcd/tree/main/contrib/raftexample

- https://github.com/etcd-io/raft
- https://raft.github.io/raft.pdf