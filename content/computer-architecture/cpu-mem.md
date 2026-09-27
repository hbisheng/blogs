---
title: "CPU <-> memory interaction"
date: "2023-02-06"
weight: 20
---

**TL;DR** CPU interacts with the main memory on the motherboard using the BUS protocol. There is a hierarchy of CPU caches (L1, L2, L3) between them.

We talked about the CPU and main memory as separate components. In this post, we'll look further into how they interact.

## CPU memory interaction

A **motherboard** is where different components of a computer are connected. The components are connected through a **bus.**

It takes time for data to be transmitted on the bus. The **bus speed** refers to the data rate that can be transferred.

### North/south bridge architecture

For easier understanding, we'll first look at a historical architecture that was well-known.

It involved a **north bridge** (also called the **memory controller hub**) and a **south bridge**.

The north bridge is for components that require the fastest processing. It is

- connected to the CPU through the **FSB (front-side bus)**
- connected to DRAM through the **memory bus**
- connected to the video card through **AGP (Accelerated Graphics Port)**

The FSB has a clock source.

- The CPU uses that clock source to generate a higher clock frequency.
- The FSB clock is used to power the memory bus; its clock rate is the frequency of the DRAM.

The south bridge connects the north bridge to other IO devices that require less priority, such as the network interface card, and the disk. Those devices usually have their own clock source.

In modern CPU architecture, the north bridge is integrated into the CPU.

The configurations on the motherboard (e.g. clock frequencies, memory controller settings) are not set in stone. They can be configured differently during the computer startup (by the BIOS program).

### Bus Arbitration

What if two components put data onto the bus at the same time?

- That would be a **bus collision**, which should be avoided as much as possible.

A **bus arbiter** may be used to decide who should have write access to the bus. That's a centralized decision-maker. For a distributed version, the CSMA/CD protocol used in Ethernet is also valid here.

### Async Bus

CPU and DRAM can have different clock frequencies and interact through the bus asynchronously.

This [reference](https://www.eventhelix.com/fault-handling/bus-cycles/) gives some idea of how that would work. In addition to the typical address, data, read/write lines, there are other control signals:

- **Address strobe**: The CPU signals that the address line now contains a valid address.
- **Data strobe**: The CPU signals that it is ready to read/write the data. In the write case, it will put the data on the data line first.
- **Data acknowledge signal**: The DRAM signals that the read/write operation is done. During a read, the CPU may now read the data line.

PCI and AGP have their own clocks. They interact with the CPU in this async fashion.

### **Sync bus**

The DRAM could connect to the same clock the CPU uses through the memory interface.

- With a synchronized bus, the CPU can predict when the data is available and grab the data blindly ([ref](https://users.cs.fiu.edu/~prabakar/cda4101/Common/notes/lecture10.html)). No need for sophisticated control signals.
  - e.g. **CL (Column Address Strobe latency)** is the delay between command and data availability. It can be 15.

## CPU cache

There are three layers of CPU caches between the CPU and the main memory.

### Cache Layers

L1 cache: 2KB to 64KB

- Instruction Cache
- Data Cache

L2 cache: 256KB

- Shared across cores. Inside/outside of the CPU.

L3 cache: 1MB-4MB

- Shared across cores. Outside of the CPU.

### Cache hardware

A **word** is the amount of data RAM operates on in one operation.

A CPU cache has multiple **cache lines**.

- Each cache line stores one **block** of data. It's the smallest unit of cache operation.
- A block contains multiple words.

In the following example, suppose that we have

- A 4GB main memory
  - **Word size** is 64 bits
- A 64KB cache
  - **block size** is 64 bytes, so the cache line size is also 64 bytes
  - Based on the block size, there are 8 words per block.
  - Based on the cache size and the cache line size, there are 210 cache lines

Let's do some simple calculations

- Each block contains 8 words
  - 3 bits for word addressing within a block
- Total capacity:
  - 4GB main memory → 235 bits
    - 232 bytes (each byte is 8 bits)
    - 229 words (each word is 64 bits)
      - 226 blocks (each block has 8 words).
  - 64KB cache
    - 210 cache lines

When we store a block in the cache, we need 26 bits in total to find it's original place in the main memory.

How that 26 bits are distributed depends on the caching placement strategy.

The start of a cache line is a **tag** used to identify a block**.**

1. **Direct Mapping**
   - Imagine the main memory is divided into chunks of cache sizes.
   - 10 bits are known based on the position in the cache line.
   - Only the first 16 bits are needed to identify a block.
   - Word address (29 bits) = tag (16 bits) + cache line position (10 bits) + word id (3 bits)
2. **Associative Mapping**
   - The position adds no information. You need the full 26 bits.
   - A block can go to any cache line. You need a linear search (hardware supported) to see if there's a cache hit for your address.
   - Word address (29 bits) = tag (26 bits) + word id (3 bits)
3. **Set associative**
   - It's like a combination of the previous two. A block can go to any cache line of its set.
     - First, do a direct mapping to get the set number and then do a search.
   - Blocks belong to alternative set IDs. The first block belongs to set 0, the second block belongs to set 1.
   - Suppose 24 sets, there will be 26 cache lines per set.
   - 4 bits are known due to which set the block belongs to.
   - Word address (29 bits) = tag (22 bits) + set (4 bits)  + word id (3 bits)

**K-way set associative**

- It's like a direct mapping, but with multiple partitions.
- You can read simultaneously to find a match.

**Block replacement**

- When there's a cache miss, which one to replace?
  - Random, FIFO, FILO

**Write Strategy**

- 1.1 Write Through. Update both the cache and the main memory.
- 1.2 Write back/deferred. Only the cache is updated (with a dirty bit)
- 2.1 Write allocate. Put the data in the cache if the data to be modified is not in the cache.
- 2.2 Non-write allocate. Skip the cache and update the main memory directly.

### Basic numbers

Some basic numbers for data access latency ([ref](https://stackoverflow.com/questions/4087280/approximate-cost-to-access-various-caches-and-main-memory))

- Registers: 1 clock cycle.
- L1 cache: 0.5ns (3 clock cycle)
- L2 cache: 7ns (9 clock cycle)
- L3 cache: (21 clock cycle)
- Memory access latency: 100ns (150-400 clock cycle)
- SSD latency: 0.05ms
- Disk latency: 5ms

## Reference

https://www.eventhelix.com/fault-handling/bus-cycles/

https://users.cs.fiu.edu/~prabakar/cda4101/Common/notes/lecture10.html

https://www.youtube.com/watch?v=7QD9fQRQ\_l0
