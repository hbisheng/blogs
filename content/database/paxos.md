---
title: "Paxos"
date: "2023-03-11"
weight: 70
---

**TL;DR** Paxos is a protocol for distributed consensus. In its simplest form, a group of participants collectively decide and agree on a single value of the system.

The problem in general was discussed in the [Raft post]({{< relref "/database/raft.md" >}}). We'll be talking about single-decree Paxos where a group of participants want to decide on a single value. A value can be considered as a log entry—an event that leads to state changes. Single-decree Paxos can be repeated to decide on multiple values, forming a log.

[This blog post](https://maheshba.bitbucket.io/blog/2021/11/15/Paxos.html) abstracts the problem of consensus as a WOR (Write-Once register). It is quite intuitive; in Paxos, once a value is chosen, it must not be changed.

## Paxos protocol

In Paxos, there are three kinds of agents: proposers, acceptors, and learners.

The algorithm involves **one or more rounds** that are driven by the proposers. Each round is identified by a **sequence number** and driven by one proposer. Multiple rounds can happen concurrently without violating safety.

### A single round

In each round, the proposer has two phases

1. **Prepare Phase** (locking phase or value determination phase)
   - The proposer sends a **Prepare request** along with its **sequence number** to all acceptors.
   - An acceptor only responds if it hasn't received a Prepare request with a higher sequence number.
   - If an acceptor responds, it will return the value with the highest sequence number, among all values that it accepted.
   - The proposer moves on to the next phase if it receives a response from the **majority of acceptors**.
   - The proposer **determines the value** to propose by picking the value with the highest sequence number from the responses from the acceptors. If there's no value accepted before, the proposer is free to propose any value.
2. **Accept Phase**
   - The proposer sends an **Accept request** along with the proposed value and the sequence number.
   - An acceptor only accepts the value if it hasn't received a Prepare request with a higher sequence number.
   - If the majority of acceptors accept the value, the mission is complete.

### Why multiple rounds?

If everything goes well, a single round will get the problem solved. But Paxos is designed to survive different failure scenarios. Nodes can die and restart at any time. The system should continue to function as long as a majority of participants are up.

A proposer can go down at any step of the process. For example, it may fail to send out enough Accept requests to all acceptors or it could be super slow as if it has crashed. Multiple rounds are needed in these failure scenarios.

The sequence number of a round **establishes precedence between rounds**. The Prepare phase of a higher-sequence-number proposer is like a fencing token that asks the acceptors to forget about previous proposers.

For any two rounds, we can consider the possible sequence of events that can happen on a single acceptor

(Px: prepare of round x; Ax:accept of round x)

1. P1 (involved in round 1; aborted)
2. P2 (involved in round 2; aborted)
3. P1A1 (involved in round 1; accepted)
4. P2A2 (involved in round 2; accepted)
5. P1A1P2A2 (involved in both round 1 and round 2; both accepted)
6. P1P2A2 (round 2 preempted round 1; round 1 aborted and round 2 succeeded)

Note that the Paxos protocol prohibits P2P1 from happening.

**Value determination of the Prepare phase**

If a value is chosen at a certain round, Paxos ensures that it is impossible for a different value to be chosen again. It does this by ensuring that **all future proposals will propose the same value**. This is the **key purpose** of the Prepare phase—picking up values that have been accepted before.

Let's say round 1 gets a value chosen. The prepare phase of round 1 must have reached a majority of acceptors. Now, for future rounds, there must be an overlap between the set of acceptors that accept the chosen value in round 1 and the acceptors that respond to the prepare request of round 2.  In other words, there must be at least one node in the P1A1P2 situation, which ensures that the proposed value is the same as round 1 in round 2.

The prepare phase ensures that any value accepted by the majority is picked up, inherited, and propagated to all future proposals. In Raft, this induction-like inheritance is done at the leader election step. The new term leader must contain any entry replicated to the majority.

You may ask if a value has been chosen, what's the need to have more rounds? Why not just stop everything? It could be that there are multiple concurrent rounds and one of the rounds chooses a value while others are in-flight. We need to handle conflicts between rounds.

What about **learners**? Whenever an acceptor accepts a value, it can send a message to a learner for it to count. Once a majority is reached, it will learn that a value has been chosen. It can then inform the proposers to stop.

## Paxos variations

The appendix in the [original Paxos paper](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf) is a good reference. That paper poses a relatively strict constraint that requires the majority sets to be the same in the two phases.

In [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf), the version of Paxos allows the majority sets to be different. To support that, it needs to allows an acceptor to accept a ballet higher than what's prepared.

In the TLA+ specification, an acceptor is allowed to accept a ballet higher than what's prepared, but when that happens, it also updates its highest ballet number so that it denies future prepare/accept requests with a smaller sequence number. To me, it's like automatically inserting a prepare request right before the accept request.

- https://github.com/tlaplus/tlapm/blob/main/examples/paxos/Paxos.tla#L109

Some claimed that [PMS has a bug](https://brooker.co.za/blog/2021/11/16/paxos.html) but I disagreed. In the failure scenario mentioned in the blog post, the accepted ballet with a lower number will not overwrite the higher ballet already accepted.

Some said PMS should have said, “if the acceptor accepts a ballet higher than what's prepared, the acceptor modifies the prepared value to the new ballot”. This is also what the TLA+ spec does. But I think this is an *optimization*, not a *requirement* for safety.

Without that optimization, consider again the sequence of events on an acceptor.

1. A1 (round 1 accepted without a prepare)
2. A2 (round 2 accepted without a prepare)
3. A1A2 (round 1 and round 2 both accepted)
4. A1P2A2 (round 1 and round 2 both accepted)
5. A2P1A1 (round 1 and round 2 both accepted)

Note that P2P1 and P2A1 are still prevented.

Now some might say that situation 5 is problematic. But I argue that it is harmless. It is only harmful if two rounds have different values and they are both accepted. But that will not happen.

- If both rounds are accepted, we just need to consider the set of nodes with A1 and the set of nodes with P2. They must have an overlap on at least one acceptor because both must have reached a majority of acceptors. Also, A1 must happen before P2 because P2A1 is not allowed. Then, P2 must have picked up the value accepted in A1. Their values must be the same.

## Paxos vs Raft

- Paxos
  - Weak leader. Having multiple leaders at the same time doesn't violate safety.
  - The two phases are like pull and push. The pull ensures the inheritance of accepted proposals.
  - Accepted by the majority makes the value final (committed).
- Raft
  - Strong leader.
  - The voting part in the leader election ensures the inheritance of committed entries.
  - Replicated to the majority with the current term makes the value final (committed).

A Raft-style Paxos description is presented in https://arxiv.org/pdf/2004.05074.pdf.

- Term: Each node uses a distinct set of sequence numbers.
- Leader election: For a particular term, only one node will campaign for the leader.
- Data flow: After the leader election, log entries can flow from followers to the leader (this corresponds to the prepare phase in the original presentation where past accepted values are picked up). Entries on the leaders can be overwritten.

## More resources

An extensive list of Paxos variations: https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-935.pdf
