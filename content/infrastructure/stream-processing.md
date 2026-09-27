---
title: "Stream Processing"
date: "2023-07-04"
weight: 40
---

**TL;DR** A stream is an unbounded sequence of events with producers and consumers. It's a rich source of data that can be utilized for diverse applications like asynchronous event processing, analytics, etc. Kafka is a prominent stream-processing service.

This post talks about a common component in the infrastructure of a company—stream processing.

## What's a stream?

A stream is an unbounded list of events. There are producers and consumers. Producers add new events to the stream and consumers read events from the stream. In most cases, the stream is **append-only** and the events are **immutable**.

Each consumer or consumer group is associated with an **offset**. The offset moves forward as the consumer acknowledges the events in the stream.

Stream processing systems accept new events from producers and **persist** them onto disk.

- One thing that makes a streaming service fast is the pattern that consumers usually consume the latest events. Since those new events are just persisted to disk, they are highly likely to still exist in the kernel page cache and can be served from memory quickly (without reading from the disk).

In several cases, the events of a stream can be **deleted**:

- For some use cases, the events are time sensitive so it makes sense to clean up events older than a certain time period.
- The stream capacity has reached, so either reject new events or delete old ones; the latter is generally preferred.

### Partition and replication

**Partition** and **replication** mechanisms are similar to a database. To achieve scalability, a stream system can partition the key space with a hash-based or range-based scheme and divide the work between partitions. For each partition, there are more than one nodes that store the data. There's usually a leader that takes in a new event and replicates that event to followers before acknowledging the clients.

### At-most/at-least/exact once semantic

A stream processing service may guarantee that an event is processed at most once, or at least once, or exactly once. The semantics are mainly determined by how you configure producers and consumers and handle message acknowledgment and retries.

#### At-most-once

Messages may be lost but are never redelivered or duplicated. This is achieved by configuring the producer to not retry failed sends (`retries=0`) and committing the offset immediately after fetching the messages in the consumer. In this setup, if processing fails or if there are issues while sending the message, the message won't be retried and hence may be lost.

In short, do not retry.

#### **At-least-once**

Messages are never lost but may be redelivered and processed multiple times, leading to duplicates. This is achieved by configuring the producer to retry failed sends (`retries > 0`) and committing the offset **after** the message has been processed in the consumer. If the processing fails, the consumer will read the message again on restart, leading to duplicates. You could handle duplicates in the downstream systems, or by making the processing idempotent, if possible.

In short, retry until success.

#### **Exactly-once**

Here, each message is delivered and processed exactly once, with no losses or duplicates. This is the most complex to achieve.

Let's try to define what it entails first.

1. For a producer, an event should be appended to a topic exactly once, without misses or duplicates.
2. For a consumer, an event should be processed exactly once, without misses or duplicates.
3. For a component that acts as both a producer and a consumer, achieve the above two.

Firstly, you need to configure the producer for idempotency (`enable.idempotence=true`).

- This setting should be preferred in general as well because the overhead is small.
- Each producer will be assigned a producer ID and each event that a producer produces is assigned a sequence number. The broker can use the combination of producer ID and sequence number to detect missing or duplicate events.

If a producer has multiple events to produce at the same time (potentially for different topics), the events are written atomically. This can be done with a two-phase commit and requires a transactional API.

On the consuming side, you need to consume the messages and commit the transactions in one go, so that if a failure occurs during processing, the read offsets are not committed, and the same messages can be read again.

## Comparison

### Stream vs Database

We are familiar with a database. A database also needs to handle an unbounded number of events/requests. How is a stream different from a database?

If we think of each event as a write, then a stream contains more information than a database: a database is only concerned with the most up-to-date value for each key while a stream stores all the history of writes. Therefore, a stream can be used to represent a database. Conceptually, a database is a compacted stream where only the latest entry for each key is kept.

The stream of replication events of a database are called Change Data Capture (CDC). The followers that consume the replication event do not blindly store the stream of events on disk. Instead, they apply the database updates from the stream onto the local copy of the database, and then the events from the stream can be discarded. Conceptually, the replication stream is a stream of state changes while each database node maintains a state machine, this view is particularly true for distributed databases.

### Stream vs PUB/SUB

How is a stream processing service different from a PUB/SUB service? PUB/SUB mostly focuses on the routing/distribution of data while a stream processing service can involve more processing. Kafka can be used for a PUB/SUB service but it can also be used for more complex processing.

## Use cases

1. Async event processing, event distribution, analytics
   - These are cases where you want to process the same events more than once.
   - (From *Design Data Intensive Application*) **Keeping systems in sync**: A system may involve many components that represent the data in different ways. A common practice is to use a database as that source of truth and other components (search index, data warehouse) act as followers of the database. The database leader establishes a total order of events within the system and streams those events to other components.
2. (From *Design Data Intensive Application*) **Immutable states at the application level:** The application state is represented and can be reconstructed by a stream of events. The events are append-only, each of which represents one transition of the application state. This is the mindset of event sourcing. The benefits include complete history and replayability.
3. Event transformation
   - **Stateful stream processing** (stream-to-stream-join)**:** Maintain a 1-hour window of search events with session IDs and match the click events by the session ID. This is a stream-to-stream join.
   - **Stream enrichment** (stream-to-table join)**:** Augmenting the stream entries with additional information from tables. The table content also comes from a stream of CDC events, whose window length is conceptually infinite.

## Kafka

### Zero-copy principle

**Background:**

- To read a file, the content must be read from the **disk** into the kernel **page cache**.
  - This can't be avoided. But the CPU is not very involved in this process due to DMA (Direct Memory Access).
- Applications run in **user space** and may have their own user-space buffer for data.
- The kernel maintains a **socket buffer** for each socket.
  - It's used for interacting with NIC when sending or receiving data.
  - It's also where the kernel TCP/IP stack wraps/unwraps the data with the protocol headers.
- To send a piece of data over the network, the data must be copied to **NIC's memory buffer**.
  - This can't be avoided. But CPU can be less involved in this process due to DMA.

**Traditional two-copy mechanism**:

1. Read from disk (disk → kernel page cache)
2. **Copy** the data to user space (kernel page cache → user space buffer)
   - Crossing the user-kernel boundary.
3. **Copy** the data to the kernel socket buffer.
   - Crossing the user-kernel boundary.
4. Send data to NIC's memory buffer.

**One-copy mechanism**:

1. Read from disk (disk → kernel page cache).
2. **Copy** the data from the kernel page cache to the kernel socket buffer.
   - Use the `sendfile()` system call in Linux, which allows data to be transferred directly from the file system cache to the socket buffer. In Java, this is the `transferTo()` method in `java.nio.channels.FileChannel`.
   - The data does not cross the user-kernel boundary.
3. Send data to NIC's memory buffer. DMA can be used to reduce the CPU's involvement.

This is the main optimization that Kafka uses to reduce unnecessary copies of data.

**“Zero-copy" mechanism**:

1. Read from disk (disk → kernel page cache)
2. NIC directly reads from the kernel page cache to NIC's memory buffer.

In theory, this would be the most efficient, but it isn't typically feasible for standard TCP/IP networking applications like Kafka. Directly copying data from the kernel page cache to the NIC's memory buffer means bypassing the kernel's socket buffer and the TCP/IP stack.
