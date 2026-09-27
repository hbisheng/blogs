---
title: "NUMA and cache coherence protocols"
date: "2023-02-13"
weight: 40
---

**TL;DR** Modern CPUs have multiple cores and multiple memory banks. Different cores have their own cache and they adopt a cache coherence protocol to maintain memory coherence.

This [series of videos](https://www.youtube.com/playlist?list=PLeWkeA7esB-OgNoVkE2lW2cVBxpDbu92h) is very helpful in building an understanding of multicore architectures.

## NUMA (non-uniform memory access)

In a modern CPU, there are multiple cores and multiple memory banks. They are connected in a way that the memory access time can vary based on the “distance” between the core and the bank. Also, some memory may be faster for loading data from disk because it's closer to the disk controller.

Each core has a corresponding local memory. One or more cores that share the memory bus and the same local memory is called a **NUMA node**. Each NUMA node has its memory controller. The OS tries to allocate memory in the same NUMA node of the running processes. Similarly, if a process needs to access the disk, it will be best to schedule it on the NUMA node that's close to the disk controller.

If one CPU wants to access the memory of another NUMA node, it relies on CPUs to relay its request. There's something similar to routing involved.

## Cache coherence protocols

**Definition of Coherence:** All caches have a consistent view of data that is lastly written at any time.

Some examples to see where problems might arise:

- If the piece of data is only cached by one core, there's no problem reading and writing it.
- If none of the cores modify the data, there's no problem.
- There's a problem when the data is present in multiple caches and one of the processors wants to modify it.

### **Snooping**

There's a shared memory bus that the caches can put data on and snoop from.

Each cache maintains a state machine of cache block and transitions the state based on CPU/bus activities.

This only works for up to 4 to 8 cores because the shared bus and full-meshed broadcasting are not scalable.

### **Directory-based**

The state of cache blocks can be stored in a centralized directory. By looking up the directory, unnecessary messages can be avoided and more efficient point-to-point communications can be used.

## Cache coherence protocols

These protocols are discussed in the context of the snooping-based mechanism. But the state transitions apply to directory-based mechanisms as well.

### **MSI protocol**

1. Invalid: you don't have a valid copy.
2. Shared: you have one copy and it's clean.
3. Modified: you have **the only copy** and it's dirty.

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
|  |  | **Invalid (I)** | **Shared (S)** | **Modified (M)** |
| **From CPU** | **Your Read** | S  (bus activity: issue read miss) | noop | noop |
| **Your Write** | M  (bus activity: issue write miss) | M  (bus activity: issue write invalidate) | noop |
| **From bus activity** | **Other's read miss** | noop | noop | S  (bus activity: need to flush to the cache&memory. will block the read or supply data to it) |
| **Other's write miss** | noop | I | I  (bus activity: need to flush to the cache&memory. will block the write) |
| **Other's write invalidate** | noop | I | Impossible because others cannot issue a write invalidate if you have the only copy. |

“noop” represents that the state doesn't change.

### **MESI protocol**

**Motivation**: there's no need to send invalidate if you have the only copy of the data. Exclusive is added to describe this state.

**States:**

1. Invalid: you don't have a valid copy.
2. Shared: you have one of the many copies and it's clean.
3. Modified: you have the **only copy** and it's **dirty**.
4. Exclusive: you have the **only copy** and it's **clean**.

MESI is also an invalidate-based protocol.

|  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- |
|  |  | **Invalid (I)** | **Shared (S)** | **Exclusive (E)** | **Modified (M)** |
| **From CPU** | **Your Read** | (bus activity: issue read miss and listen to all caches)  S (there's other reader) or E (no other reader) | noop | noop | noop |
| **Your Write** | M  (bus activity: issue write miss) | M  (bus activity: issue write invalidate) | M   (no need to send invalidate) | noop |
| **From bus activity** | **Other's read miss** | noop | noop | S | S  (bus activity: need to flush to the cache&memory. will block the read or supply data to it) |
| **Other's write miss** | noop | I | I | I  (bus activity: need to flush to the cache&memory. will block the write) |
| **Other's write invalidate** | noop | I | Impossible | Impossible |

### **MEOSI protocol**

**Motivation**: Defer writing back the value to the main memory. The modified state doesn't allow other readers; the owned state does.

1. Invalid: you don't have a valid copy.
2. Exclusive: you have the only copy and it's clean.
3. Modified: you have the only copy and it's **dirty**.
4. Shared: you have one of the many copies and it's clean.
5. Owned: you have one of the many copies and it's **dirty.**

|  |  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- |
|  |  | **Invalid (I)** | **Shared (S)** | **Exclusive (E)** | **Modified (M)** | **Owned (O)** |
| **From CPU** | **Your Read** | (bus activity: issue read miss and listen to all caches)  S (there's other reader) or E (no other reader) | noop | noop | noop | noop |
| **Your Write** | M  (bus activity: issue write miss) | - M (bus activity: issue write invalidate) - O (bus activity: issue write update) | M  (no need to send invalidate) | noop | noop |
| **From bus activity** | **Other's read miss** | noop | noop | S | - S (bus activity: need to flush to the cache&memory. will block the read or supply data to it) - O (bus activity: supply the data to the requesting cache) | noop  (supply the data to the requesting cache) |
| **Other's write miss** | noop | I | I | I  (bus activity: need to flush to the cache&memory. will block the write) | I |
| **Other's write invalidate** | noop | I | Impossible | Impossible | I |
| **Other's write update** | noop | noop | Impossible | Impossible | S |

### MESIF protocol

One problem with the MESI protocol is that multiple readers may reply with the data which is redundant and inefficient.

A Forward (F) state can be dedicated to describing a responder that should supply data to the new reader.

F is usually the most recent reader of the cache (without the F state it would be Shared)

## Cache coherent NUMA

A more scalable architecture is that there are multiple page directories that are stored alongside the cache within a NUMA node.

Different nodes may communicate to ensure the state machines of the cache lines are maintained.

When there is a request, there may be three kinds of nodes:

1. Local node: The node making the request.
2. Home node: The node that holds the main memory for the cache block and the directory.
3. Remote node: Another node that holds a copy of the cache block.

Consider a very simple example:

The remote node holds a dirty copy of the data. The local node requests a write of the data. Here are the events:

1. The local node sends a write miss to the home node.
2. The home node checks the directory and finds the dirty copy.
3. The home node sends an invalidate request to the remote node.
4. The remote node sends back the latest data and invalidates the cache.
5. The home node gets the data and updates the main memory.
6. The home node supplies the data to the local node for it to write.

## References

https://www.youtube.com/playlist?list=PLeWkeA7esB-OgNoVkE2lW2cVBxpDbu92h
