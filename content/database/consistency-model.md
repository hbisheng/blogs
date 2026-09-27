---
title: "Database Consistency Model"
date: "2023-03-20"
weight: 10
---

**TL;DR** Consistency is a problem when your system stores multiple copies of the same data. Consistency levels allow us to reason about the guarantees provided by a database system and what outcomes are allowed/forbidden.

## Database Consistency Model

A database is a system where you store data and retrieve it later. A traditional database is merely a process on a single computer and there's only a single copy of the data. In modern days, to scale up performance and achieve failure tolerance, data is replicated to multiple machines and stored as multiple copies. Each copy is called a replica. Ideally, each copy of the data is exactly the same but that's hard to achieve in practice, hence the problem of consistency. Some database systems also explicitly trade consistency for higher availability. Understanding the consistency models is helpful for reasoning about modern databases and using them correctly.

It's not very straightforward to reason about all the consistency levels. We'd like to discuss them with some concrete examples. To do that, we need to clarify the environment where the examples will come from.

In this environment,

- There's only a single object
  - The data has multiple replicas that are stored on different database nodes.
- There are multiple processes that can read or write the data
  - Each process does operations in series.
  - Different processes may do operations concurrently.

Imagine there's no consistency guarantee, then any operation by any process can be served by any replica, which will of course cause all kinds of weird behaviors (e.g. you may not read the value you've just written).

**Key definitions:**

- **Session order**: The order of operations within a process.
  - If a process first does a write W1 and then a read R1, the session order is W1→R1.
- **Visibility order**: Whether a write is visible to a read.
  - In the above case, W1 may not be visible to R1, in which case R1 will read nothing. How come? We can imagine a database implementation where W1 is sent to replica A (and it's supposed to be replicated to all other replicas) but R1 is performed on replica B (before the write arrives).
- **Arbitration order**: The order in which write conflicts are resolved. This determines the value of a read when multiple writes are visible.
  - If process A does [W1, R1] and process B does [W2]. Suppose that both W1 and W2 are visible to R1, which one should R1 return? If the arbitration order is [W2,W1], then R1 should return the last observed write, which is W1.

## Consistency Levels

These are the consistency levels we will talk about.

1. **Read Your Writes**: Within a process, if a write happens before a read, the write is visible to the read.
2. **Monotonic Writes**: If a process first do w1 and then w2, every process observes w1 before w2.
3. **Monotonic Reads**: For any three operations w, r1, r2. If w is visible to r1 and r1 happens before r2, then w is visitable to r2.
4. **PRAM:** The combination of the above three (Read Your Writes + Monotonic Writes + Monotonic Reads). Real-time order within a process is respected. Cross-process write orders are preserved (thus the name *pipeline*).
5. **Writes Follow Reads:** If a write by any process is observed by a read in one process which later does a write, the second write must be visible after the first write. This case establishes a causal relation.
6. **Causal consistency**: In addition to session orders in PRAM, some additional causal (happen-before) relationships are respected.
7. **Sequential consistency**: Same total order across all processes while respecting in-process return-before constraints (as in PRAM).
8. **Linearizability:** Same total order across all processes while respecting all return-before constraints. The system behaves as if each operation is executed instantaneously at a certain point in time (called its linearization point) between its invocation and completion. For a single read-write register, it's the same as being **atomic**.

In the following table, we will mention some concrete examples that are allowed or forbidden by a certain consistency level. A system meets a certain consistency level when its **observable outcomes** obey the constraints imposed by the consistency level. Here is the example scenario we'll use across the discussion:

```
# Process1: W1      W2  R1
# Process2: W3  R2  W4
# Process3:                R3  R4
```

|  |  |  |
| --- | --- | --- |
| **Level** | **Example Constraints** | **Possible Implementation** |
| No consistency | It's possible that R2→∅ | A read/write is routed to any replica with async replication. |
| Read Your Writes | It's required that R2→W1 or R2→W3 | Only read from replicas that you have written to. |
| Monotonic Writes | It's impossible to get (R3→W4 and R4→W3). | Ensure that writes of the same process are replicated to others in order. |
| Monotonic Reads | It's impossible to get (R3→non-∅ and R4→∅). | Read from the same replica all the time (sticky client) |
| PRAM | It forbids all examples prevented by the above three consistency levels.  Different processes may observe writes in different orders (while session orders are guaranteed). For example, it's possible that R2→W1, R3→W4, R4→W1. | Read from the same replica all the time, which has to be the ones where you write your values to. Writes of one process are replicated in order (although no total order is required). |
| Writes Follow Reads | This consistency level prunes certain sequence of writes for a certain read, but by itself doesn't prune any observable outcome.  What may be interesting is that, the following outcome is possible with PRAM but not with **PRAM+WFR**: R2→W1, R3→W4, R4→W1 (Once R2 observes W1, W1 must be observed before W4) | Writes need to be coupled with some metadata to establish precedence over previous writes. |
| Causal Consistency | It's impossible to get R2→W1, R3→W4, R4→W1 (same example as above). | When you read something, you get a vector timestamp, you update your timestamp before writing a new value. You'll be able to tell whether two writes are concurrent or one is causally before the other. |
| Sequential Consistency | It's impossible to get R1→W3, R2→W1 (the order of W3 and W1 is not consistent across two processes) | Use Lamport timestamp to generate a total order of events and use that total order to resolve conflicts. |
| Linearizability | It's impossible to get R3→∅ (the return-before relationship needs to be respected). | Use a leader to serves all reads and writes while having followers to maintain fault tolerance.  Note that if async replication is used and reads can be served by the follower, the system can return **stale data,** which violates linearizability. But sequential consistency can still be maintained in this case. |

## References and Resources

- [Consistency in Non-Transactional Distributed Storage Systems](https://arxiv.org/pdf/1512.00168.pdf)
  - The paper that formally represents non-transactional consistency level
- http://cs.boisestate.edu/~amit/teaching/555/handouts/replication-and-consistency-handout.pdf
