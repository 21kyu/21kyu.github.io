---
title: Basic ETCD
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2022-01-19 23:00:00 +0900
categories: [kubernetes, analyze]
tags: [kubernetes, watch, event]
math: true
render_with_liquid: false
---

* Highly Available
* Fully Replicated
* Strong consistency model
* Scalable watch mechanism
* Concurrency control primitives

Distributed Key/Value store
Distributed consensus is a fancy way to express the idea of getting more than one party to agree on something

* We use tools like Kubernetes because we want to build a fault tolerance system, which requires redundancy and multiple resources
* Taking action on the system means that every manager entity in the system has to agree about the action

## What is Raft

In a distributed model, that log entry can only "become truth" once the manager agree on the new value.

Consensus Datastore: RAFT Consensus Algorithm

* Raft is an algorithm to manage consensus-based systems (like container orchestrators)
* It is designed to be easy to understand
* It also has a cute logo

Raft implements consensus by a leader approach.
The cluster has one and only one elected leader which is fully responsible for managing log replication on the other servers of the cluster.
It means that the leader can decide on new entries' placement and establishment of data flow between it and the other servers without consulting other servers.
A leader leads until it fails or disconnects, in which case a new leader is elected.

### Raft is used in a lot of places

Orchestration systems typically use a key/value store backed by a consensus algorithm

* Kubernetes -> etcd, and by extension, every system that uses etcd
* Docker (swarm mode)
* Nomad
* Zookeeper uses Zookeeper Atomic Broadcast (ZAB), which is similar to Raft

#### Raft is responsible for

[The Secret Lives of Data](http://thesecretlivesofdata.com/raft/)

Leader election
: Raft uses a randomized election timeout to ensure that split vote problems are resolved quickly.
This should reduce the chance of a split vote because servers won't become candidates at the same time: a single server will time out,
win the election, then become leader and send heartbeat messages to other servers before any of the followers can become candidates.
[demo.consensus.group](https://raft.github.io/raftscope/index.html)

Log replication
: The leader is responsible for the log replication.
It accepts client requests. Each client request consists of a command to be executed by the replicated state machines in the cluster.
After being appended to the leader's log as a new entry, each of the requests is forwarded to the followers as AppendEntries messages.
In case of unavailability of the followers, the leader retries AppendEntries messages indefinitely, until the log entry is eventually stored by all of the followers.
[logging-post](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)

Safety
: Election safety (_at most one leader can be elected in a given term._),
Leader append-only (_a leader can only append new entries to its logs (it can neither overwrite nor delete entries)._),
Log matching (_if two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index._),
Leader completeness (_if a log entry is committed in a given term then it will be present in the logs of the leaders since this term_),
State machine safety (_if a server has applied a particular log entry to its state machine, then no other server may apply a different command for the same log._)

#### Restore from a backup

You have a backup, right? : Some tools manage this for you

Generally speaking:
* The log snapshots live in `/snap`
* The WAL lives in `/wal`

### Quorum

The minimum number of votes needed to perform an operation.
Without quorum, your system can't do work.

Quorum is satisfied with over 50% of agreement.

$$ (N/2) + 1 $$

| Managers | Quorum | Fault Tolerance |
|----------|--------|-----------------|
| 1        | 1      | 0               |
| 2        | 2      | 0               |
| 3        | 2      | 1               |
| 4        | 2      | 1               |
| 5        | 3      | 2               |
| 6        | 4      | 2               |
| 7        | 4      | 3               |

### Key Space

`/prefix/K8S type/namespace/resource name`

```shell
/registry/pods/production/service1-xxxxxxxx
/registry/replicasets/development/service1
```
{: .nolineno}

_lexically ordered_

### Request/Response Operations

```shell
RANGE <start_key>..<end_key>
PUT <key> <value>
DELETE RANGE <start_key>..<end_key>
TXN (if <condition> then <op1, ..> else <op2, ..>)
```
{: .nolineno}

_Linearizable Consistency (a.k.a External Consistency)_

### Streaming Operations

```shell
CREATE WATCH <key1>..<key2>
client -------------------------------------------> etcd
                gRPC bidirectional stream
client <------------------------------------------- etcd
WATCH CREATED <watch-id>
EVENT <watch-id> PUT <key1> <value1>
EVENT <watch-id> DELETE <key2>
...
```

Eventual Consistency

client <Cache>  <---- Watch stream <---- Follower <--RAFT Chatter--> Leader

### Data Storage

Multi-version concurrency control / Copy-on-write for all modifications

etcd - MVCC keyspace.
Values may be accessed by key+version.
This is used to implement the watch operation.

Compaction applies to the etcd keyspace

* Removes all versions of objects older than a specific revision number.
* Kubernetes default policy: all data older than 5 minutes every 5 minutes
* Kube-apiserver requests compactions. etcd auto-compaction is disabled.

BoltDB - MVCC internally enable 1 write + N reads to be executed concurrently.

Defragmentation applies to the bolt db file

* Recovers all free space in the bolt db file.
* Only to shrink a db file as bolt does not automatically shrinks it`s file.
* Etcd will defrag and the file only if requested. This is a "stop-the-world" operation.

### How etcd Stores and Serves Data

For each write:
1. Append write to WAL (Write Ahead Log (.wal files)
2. Apply write to Keyspace (Persisted Keyspace (db file)

Every "--snapshot-count" writes:
* Create a snapshot file
* Record revision snapshot was created to WAL
* Remove WAL file older than the snapshot

RAFT ensures WAL log is the same on all members of an etcd cluster!
