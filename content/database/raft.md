---
title: "Raft"
date: "2023-03-11"
weight: 60
---

**TL;DR** Raft is a protocol for distributed consensus. It's a strong leader protocol where the leader forces the followers to have the same log entries. Only an up-to-date candidate can be elected which guarantees that a committed entry is contained in all future leaders.

## Distributed consensus

Raft is a protocol for **distributed consensus**. Distributed consensus is about making all participants in a distributed system agree on certain things. Usually, each participant maintains a state with variables and there are events that lead to state changes. The sequence of events forms a log.

These participants collectively form a system that serves certain purposes. We want to keep all logs in sync. This requires establishing a total order of events.

Why don't we just **make one of the** nodes **the leader** and let it decide everything? That's a good idea, but which node?

Let's say we just **hardcode** one node as the leader. But if that node fails, everything is stalled; no progress can be made. The system has to sacrifice availability (liveness) to achieve correctness (safety).

## Semi-sync

Let's say we have **an external system that picks the next leader**. We need to ensure that such a failure event does not violate the contract between the client and the system. For example, the client expects every write acknowledged by the system to be available for reads again.

What about a semi-sync approach? The leader requires every data update to be replicated to one of the followers before acknowledging the client. If there is a leader failure, the external system needs to promote the follower with the most recent data. This is good overall but such a system doesn't solve the problem of distributed consensus:

- You need to have some mechanism to ensure that *the external system* is always available; a leader election mechanism is also needed there so that there should be only one node that is taking action. You can't keep deferring the problem to another external system; it has to be solved somewhere.
- You also need a source of truth to put the decision of a leader. It could be yet another system or it could be the same system.

Another problem is that such a problem can only tolerate a single node failure.

- In a 3-node scenario, semi-sync already replicates the data to a majority. Tolerating a single-node failure is the best you can do.
- In a 5-node scenario, it is possible to tolerate two node failures. In general, we can build a protocol that tolerates the loss of any minority of the nodes.

## Raft protocol

Raft ([paper](https://www.usenix.org/system/files/conference/atc14/atc14-paper-ongaro.pdf)) solves the above problem

- It is equipped with a **leader election** mechanism.
- It is **fault-tolerant**, being able to tolerate node failures up to (but not including) the majority of nodes.
- Participants collectively decide what the **next state transition** is.
  - Different nodes may have different progress in terms of log replication, but each node is able to tell whether a log entry is final and thus safe to apply to its states.

### Leader election

In Raft, a node can have one of the three states: **follower**, **candidate**, **leader**.

When there's no leader or the leader fails, followers turn themselves into candidates and launch campaigns to become leaders.

Raft has a concept called **term**, whichis a monotonically increasing number. **There's at most one leader per term**.

A term corresponds to a certain time period. A term always starts with an election.

Each node keeps track of the current term, updating its term when another node informs it of a higher term.

**Leader election process**

- During a leader election, a candidate increases its current term by one and **requests votes** from all nodes. If it collects votes from a majority of nodes, it turns itself into a leader.
- A node can vote exactly once per term. **It votes for the first candidate that is at least as up-to-date as itself**. This is a key constraint that prevents a candidate without the latest committed entries to be elected as the leader.

What if there are multiple candidates at the same time? There could be split votes and none of the candidates wins. Raft introduces a randomness factor to the election timing to reduce such conflicts. Eventually, one candidate should win.

Term is also useful for evicting stale leaders. A leader of an old term turns itself into a follower when encountering a higher term.

### Log Replication

Raft is a strong leader protocol. Log entries flow in one direction; only a leader can send log entries to followers. This simplifies the protocol. The leader forces the followers to duplicate its logs.

Only a leader processes request from clients. When there is an update, the leader adds a new entry to its log with its term. An entry is considered committed (final) when

- It belongs to the current term of a leader
- It is successfully replicated to a **majority** of nodes (including the leader)

### Correctness of Raft

Raft maintains certain properties

**Leader Completeness:** A leader must contain all committed entries.

- Leader Completeness is ensured by the voting constraint mentioned above.

**Log matching:** If a log entry at one position matches on logs on two nodes, all previous entries match as well.

- Log matching can be achieved by ensuring that an entry is only appended if its previous one matches.

**State Machine Safety**:If any server has applied a particular log entry to its state machine, then no other server may apply a different command for the same log index.

- An entry is only committed after it is replicated to the majority of nodes. Future leaders must be up-to-date, which means they will contain this committed entry in their log and will commit the entry as well eventually.

### Failure scenarios

What if the leader commits a log entry and then dies immediately?

- An entry is only committed after it is replicated to the majority of nodes. Only an up-to-date candidate can be elected the leader. The new leader is guaranteed to contain this committed entry.

Is it possible for an entry to be replicated to the majority but never committed?

- Yes. If an entry is not replicated to the majority **with the highest term**, it is not guaranteed to be picked up by the next leader. The next leader can win by **ignoring** that entry with a higher-term entry.
- There's one scenario in the paper that involves three terms
  - A term-1 entry that is not fully replicated.
  - A term-2 leader which does not have the entry inserts a diverging entry with term 2 at the same position.
  - A term-3 leader containing the term-1 entry replicates it to the majority.
  - Note that at this point, nodes with the term-1 entry may still vote for the term-2 leader because the diverging entry with term 2 wins.
  - If the term-2 leader becomes a leader again (with term 4), the term-1 entry will be overwritten.
