---
title: "Memory Management"
date: "2023-02-20"
weight: 70
---

**TL;DR** The OS is responsible for managing the memory, including allocation, deallocation, defragmentation, and reclaim.

## Memory allocation

**Buddy system**

- Maintain a list of free blocks.
- Allocation
  - Choose the smallest possible block during allocation. Split the memory block if the allocation takes less than half of the block. Say you have one block of 16KB and the user requests a block of 4KB, you end up with two free blocks: 4KB and 8KB.
- Deallocation
  - When a block is freed, if its buddy (the node that shares the same parent) is also free, they can be merged into a larger free node.

The buddy system reduces fragmentation to some degree. At least smaller blocks can be merged at times to serve larger requests.

## Memory defragmentation

Let's first see how memory can get fragmented. Say you are managing 16K of memory

|  |  |
| --- | --- |
| Initial state | <-----------------------16K-----------------------> |
| Allocate 3 \* 4K | <-----------------------16K----------------------->  <----------8K-----------><-----------8K---------->  <----4K----><----4K----><----4K----><----4K---> |
| Release the second block | <-----------------------16K----------------------->  <----------8K-----------><-----------8K---------->  <----4K----><----4K----><----4K----><----4K---> |
| Allocate 1 \* 8K | Request can't be fulfilled unless the allocated memory gets defragmented. |

The **memory compaction** process:

- The OS proactively moves memories around to reduce fragmentation. This has to be coordinated with the user processes by changing the page tables. The TLB needs to be invalidated or the page fault handler needs to be invoked so that the old address is forgotten.

## Memory reclaim

All memory allocated by the processes is supposed to be used; the OS can't simply take them away. What the OS can do is offload some pages onto the swap area of the disk. This is done by a **memory reclaim** process. When the program requests the data of a swapped page, the OS can swap it back into memory.

But if the program constantly requests them back and fights against the reclaim process, this is called **page** **thrashing** which slows down the entire system.

**Watermarks**

- High: When the available memory ≥ the high watermark, the reclaim process goes to sleep.
- Low: When the available memory ≤ the low water mark, the reclaim process runs in the background.
- Min: When the available memory ≤ the min watermark, the reclaim process runs synchronously (in a blocking fashion).

Check out the diagram in <https://www.kernel.org/doc/gorman/html/understand/understand005.html>

### vm.watermark\_boost\_factor

The `vm.watermark_boost_factor` setting can cause more aggressive memory reclaim behavior. By tuning up the watermarks, it is meant to claim more memory proactively with fewer attempts, which *might* improve performance.

**Problems:**

- It could try to offload too much memory onto the disk and the disk IO cannot keep up.
- It could lead to the stealing of pages that are still actively used by the processes and the processes will fight against it by requesting the memory back (page thrashing).

Setting it to 0% disables the feature.

## More resources

https://www.kernel.org/doc/gorman/html/understand/
