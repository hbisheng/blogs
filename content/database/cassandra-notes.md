---
title: "Cassandra notes"
date: "2023-05-22"
weight: 90
home_group: "notes"
---

Cassandra is a leaderless distributed key-value database.

Cassandra does not support **secondary indexes**. It does support **range queries** for data on a particular node (things are always easy when the data is colocated; it's hard when the data is distributed).

### Sharding

It uses **consistent hashing** to shard the data across multiple nodes.

- Each node has a certain place on the hash ring. It's responsible for a certain hash range. When `vnode` is enabled, each node is mapped to multiple locations on the hash ring for better data distribution.
- With a key at hand, you map it onto a value on the hash ring and go clockwise (or counter-clockwise, depending on the implementation) to find the first node on the ring. That's the node where the key will be stored.
- Each key may be mapped multiple times onto the hash ring and be stored in multiple nodes for redundancy.

Primary key contains a clustering key. The clustering key can be used to group relevant data together.

### Replication

Clients can specify how many nodes have to ack the request before the server can return.

**Quorum** read/write is a common setup that claims to achieve “strong consistency”. I'm skeptical. Here's a scenario where a failed write eventually made it into the system; it's confusing, if not wrong. Maybe this is called “eventual consistency”.

Scenario: Let's say we have 5 nodes and require 3 nodes for read/write acknowledgment.

|  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- |
|  | Node A | Node B | Node C | Node D | Node E |
| Operation 1: Write(X → v1)  It fails to reach three nodes. | ✅  (no rollback) | ✅  (no rollback) | ❌ | ❌ | ❌ |
| Operation 2: Read(X) → v0 |  |  | ✅ | ✅ | ✅ |
| Operation 3: Read(X) → v1  Read repair happens. |  | ✅ |  | ✅  (Read repair v0 → v1) | ✅  (Read repair v0 → v1) |

### Lightweight transactions (LWT)

Lightweight transactions can be used on compare-and-set operations. In short, when you specify an IF condition on your updates, like inserting an entry if it hasn't existed, it becomes an LWT. It's like a write with preconditions.

To handle an LWT, the coordinator node uses Paxos to reach a consensus across relevant nodes: the preconditions need to be checked before the writes can be applied. This to some degree maintains the linearizable guarantee for applicable requests.

### JMX port

Cassandra, as a Java application, exposes a port (called the JMX port) for certain administrative work, like checking the cluster status and triggering certain operations.
