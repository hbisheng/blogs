---
title: "What I know about Redis"
date: "2023-04-08"
weight: 50
---

**TL;DR** Redis is an in-memory data store that supports many kinds of data structures.

Redis is a datastore written in C. It is a TCP server that waits for client requests.

Redis offers multiple data structures for storing the data, including string, list, hash map, and sorted set. The APIs are rich and powerful which enables all kinds of use cases. Redis is a good fit when your data fits into memory and **durability is not required**. Redis provides speedy access to the data structures, and manages them with bounded memory, allowing you to configure an eviction policy, e.g. LRU, LFU. **Caching** is a good use case and probably the most common one.

## Redis Specification

### Standalone mode

In this mode, Redis is not aware of other nodes (except its followers). It assumes that it owns the entire key space. It simply handles the requests and stores the data in itself.

#### Sharding

To horizontally scale it out, some external sharding mechanism can be applied. For example, it can be done via a proxy (e.g. Twemproxy) or a shard-aware client.

- Twemproxy, as an example, uses **consistent hashing** to decide the data sharding. In short, one or more hash values can be calculated for each Redis server, and each of those values maps to a location on a hash ring. For a certain piece of data, you find the corresponding server by calculating the hash value and finding the first server in a certain direction (either clockwise or counter-clockwise). Not much consistency can be guaranteed. You may not read your own write and you may read data that you think you've deleted/overwritten. The only legitimate use case is caching.

#### Replication

1. **Full sync**: When a new replica is needed, it sends a `SYNC` command to the primary. The primary assigns the new replica a unique **replication ID** and starts generating an RDB snapshot. It also maintains a **replication buffer** for the ID that stores all serialized commands that need to be sent to the replica. A **replication offset** is defined as the number of bytes that have been sent to the replica, which is only meaningful to that particular ID.
2. **Partial sync**: If the replica stops replicating briefly and tries to catch up, it may use a `PSYNC` command to continue the replication. It's possible that the replica has lagged behind too much, and the primary will request the replica to do a full sync.

In cluster mode, the same replication mechanism is used between a primary and its replicas.

#### Multi-command

**Pipelining**

Instead of sending one command and waiting for a response, the client could send multiple commands in one go and receive multiple responses. This would save network round trips and improve overall latency (but not for each individual command, as certain individual commands could be slower due to waiting).

**Pipelining vs mget/mset**

- Pipelining offers more flexibility as you can mix commands like SET and EXPIRE.
- If there is a proxy involved, pipelining is generally preferred because it is non-blocking, which allows other commands to interleave.
- If latency is the only concern, mget/mset is better.

#### **Lua script**

Useful for achieving atomic operations. Similar to transactions.

A LUA script is a string with variables in it. Why does it look like this?

1. Each native command has a syntax that makes keys and arguments explicit. So it's good for a Lua script to follow the same syntax.
2. When sharding is concerned, the keys definitely play an important role as it's involved in data routing.

### **Cluster mode**

<https://redis.io/docs/reference/cluster-spec/>

The cluster-mode functionalities are added on top of the standalone ones.

There's a dedicated cluster bus port (separate from the data port) for communicating cluster messages.

In the cluster mode, the total data space is divided into 16384 data slots. Each key maps to one of the data slots.

A node can be either a **primary** or a **replica**.

- A primary may own a certain number of slots.
- A replica replicates data from a certain primary.

#### **Basic cluster operations**

- Cluster form/creation:
  - A cluster topology needs to be decided.
    - Roles need to be assigned: Which nodes will be primaries and which will be replicas.
    - Slots need to be distributed: Which primaries will own which slots.
- Cluster add:
  - A new node joins the cluster as a new primary. Initially, it owns no slots and has no data.
  - Slots can be moved from one primary to another.
  - Slot movement protocol:
    - Move all keys first without moving the slots. Slots are marked as **exporting** on the source nodes and marked as **importing** on the target nodes.
    - A read request may need to consult both the old and the new node when its slot is being migrated.
    - After all keys are moved, the slots can be confirmed on the new node and deleted from the old node.
- Cluster remove:
  - The command needs to be sent to all members of the cluster. There's a ban duration that prevents the same node from being added back. A longer duration is useful for avoiding ghost nodes.

#### **Gossip protocol**

- Each node constantly pings other nodes to communicate the health of the cluster. There's randomness involved to reduce the total number of communications.
- A node is considered potentially failed if the regular ping doesn't go through. The timeout is called `NODE_TIMEOUT`
- A node is considered failed if the **majority of nodes** agree that it is potentially failed.
- `cluster-require-full-coverage` controls whether the entire cluster is down when one data slot is unavailable (the primary and all its replicas are down)

#### **Leader election process**

- Each node maintains its own epoch and the max epoch it has seen. Its epoch is only incremented when there're slots assigned to it or when a leader election happens.
- When there's a leader failure, the followers will initiate a **leader election** based on its rank and some randomization factor.
- A follower increments its `current_epoch` and tries to collect **enough votes** from the cluster members. It turns itself into a leader if it receives votes from the majority. If it fails, it will retry later with another new epoch.
  - The primaries ensure that they only vote once per epoch number.
  - The primaries throttle their vote to avoid having multiple followers elected (even if they have different epoch numbers)
  - Progress is “ensured” by the random election timeout.
- In what situation can there be two conflicting leaders with the same epoch?
  - A manual failover where an agreement from the majority is not obtained.

## Redis Implementation

### **IO multiplexing**

A typical TCP server listens to a port and starts a new thread whenever there is a new connection. The core event loop of Redis is single-threaded; it doesn't follow that model.

How does it handle multiple TCP connections concurrently? **IO multiplexing**. It allows you to monitor multiple sockets simultaneously.

Multiple events can happen at the same time. IO multiplexing serializes the concurrent events; it sends those events to the dispatcher one by one; the dispatcher executes the handler function one by one.

Take one multiplexing library "epoll" as an example, the user program calls some system library(EPOLL\_CTL\_ADD) to add a socket to the list of sockets being monitored, and then uses another system library epoll\_wait to poll the socket events until a timeout. EPOLL\_CTL\_DEL is used to remove a socket from the list.

This is possibly changed in Redis 5.0 where it introduces multi-threaded IO.

### **Events and handlers**

Redis is an event-based system. There are file events and time events.

- Connection request (which is also a readable event of the listening socket): create a socket for the connection; register the socket readable event to the "read processing" handler.
- Socket Readable: Do one read(); if there is no more data to read, parse the request and process it right away; Write the result to the client buffer; register the socket writable event to the "write processing" handler.
- Socket Writable: Write the result to the socket.

There is a **dispatcher** between events and handlers. It matches the event with the right handler.

### **Approximated LRU and LFU eviction policies**

<https://redis.io/docs/reference/eviction/>

Redis implements approximate algorithms for LRU/LFU

1. LRU. Typically you maintain a last access time for every key and pick the oldest one. But to avoid the sorting process (or a double linked list that keeps items sorted), Redis just samples a few keys and evict the oldest one in that sample. It turns out to work reasonably well in practice.
2. LFU. Typically you maintain a counter for every key. But to save memory, a Morris counter is used, which involves a probabilistic algorithm where we probabilistically increment a counter that represents the exponent component only.
