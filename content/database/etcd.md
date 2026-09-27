---
title: "Etcd"
date: "2023-03-12"
weight: 80
---

**TL;DR** Etcd is a distributed MVCC key-value store that implements the Raft protocol. It's wildly used in distributed coordination, such as storing system configurations (e.g. Kubernetes).

## Background

<https://etcd.io/docs/v3.5/tutorials/>

With Raft, etcd maintains a replication log where each entry is a mutative operation. The operations are committed to Boltdb, a persistent storage engine using B+ tree. The revision concept in etcd:

- **Revision**: A global monotonic number. Any update to the key space increment the revision. **Equivalent to the Raft log index**.
- **Create revision (per key):** The revision of etcd when the key is created.
- **Mod revision (per key):** The revision of etcd when the key is last modified.
- **Version (per key)**: Starts from 1. Each update increments the version by 1. A deletion reset the version to 0.

## Etcd APIs

Let's think about the implementation of those commands listed in the appendix.

`put` and `get` are straightforward

- Each `put` corresponds to one log entry in the Raft log.
- `get` is evaluated at a certain position of the Raft log.

### Transactions

Operations are serialized at the Raft leader. The conditions (read operations) will be evaluated at the current log position. All write operations in a transaction can become a single entry in the log.

### Watch

`watch` enables a client to subscribe for changes to certain keys. etcd probably maintains a list of keys (or ranges) that are being subscribed and their corresponding sessions. If a new update touches the key, the corresponding clients will be notified.

You can watch the changes of a certain key from now on or from a certain revision. etcd simply sends the data you requested and adds you to the subscribers.

### Lease

Each lease is simply a key within the key space which records a server-side timestamp. When it expires, the lease will be deleted. A lease can be extended before its expiration.

A lease can be attached to a number of keys. The existence of those keys will depend on the lease.

### Mutex

Concurrency API:

- <https://pkg.go.dev/go.etcd.io/etcd/clientv3/concurrency>
- <https://etcd.io/docs/v3.2/dev-guide/api_concurrency_reference_v3/>

Each mutex is identified by a name.

When a client tries to acquire a mutex, it creates (or reuses) a key by concatenating the mutex name and the lease of the session.

- https://github.com/etcd-io/etcd/blob/main/client/v3/concurrency/mutex.go#L113

When there are multiple clients, the client whose key has the smallest create revision is considered to hold the mutex.

If a client doesn't acquire the lock, it will wait for all client keys with smaller create revisions to be deleted.

### Leader election

A client proposes a value to be the leader of an election. When there are multiple proposals by different clients, the proposal with the smallest create revision wins. The `Observe` API allows you to see the proposal that was proposed. It can be used after winning an election.

Code example: <https://pkg.go.dev/go.etcd.io/etcd/clientv3/concurrency#example-Election.Campaign>

## Raft library

- <https://github.com/etcd-io/raft>
- <https://pkg.go.dev/go.etcd.io/etcd/raft/v3>

Will try to find some time to dig into the Raft library.

The library implementation is also used by Cockroachdb. Someone reported a [negative experience](https://www.youtube.com/watch?v=c2RyuTyVHxE) using the library though.

## Boltdb

- <https://github.com/etcd-io/bbolt>
- <https://pkg.go.dev/go.etcd.io/bbolt>

A Go library that stores a persistent B+ tree. It is where changes are committed. See the appendix for its APIs.

Will try to find some time to dig into it.

## Appendix

### Etcd API

Basic operations

```
etcdctl --endpoints=$ENDPOINTS member list

etcdctl --endpoints=$ENDPOINTS put web1 value1
etcdctl --endpoints=$ENDPOINTS put web2 value2
etcdctl --endpoints=$ENDPOINTS put web3 value3

etcdctl --endpoints=$ENDPOINTS get web2
etcdctl --endpoints=$ENDPOINTS get web --prefix

etcdctl --endpoints=$ENDPOINTS del web3
etcdctl --endpoints=$ENDPOINTS del web --prefix

etcdctl --endpoints=$ENDPOINTS watch stock1
```

Transaction API

```
etcdctl --endpoints=$ENDPOINTS txn --interactive
// condition, operation if success or failure
```

Lease related operations

```
etcdctl --endpoints=$ENDPOINTS lease grant 300
# lease 2be7547fbc6a5afa granted with TTL(300s)

etcdctl --endpoints=$ENDPOINTS put sample value --lease=2be7547fbc6a5afa
```

### BoltDB API

Source: <https://pkg.go.dev/go.etcd.io/bbolt#section-readme>

```
db, err := bolt.Open("my.db", 0600, nil)

// read-write transaction
err := db.Update(func(tx *bolt.Tx) error {
	...
	return nil
})

// read-only transaction
err := db.View(func(tx *bolt.Tx) error {
	...
	return nil
})

// update key-value pairs
db.Update(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket"))
	err := b.Put([]byte("answer"), []byte("42"))
	return err
})
```

## More resources

<https://jepsen.io/analyses/etcd-3.4.3> (such an amazing analysis)
