---
title: "Paging, Page Cache"
date: "2023-02-20"
weight: 60
---

**TL;DR** Data is moved between disks and memory in pages. The OS maintains the illusion that each process has its own virtual memory space and translates a virtual address into a physical one with the help of the page directory and page tables. The OS has kernel-level page caches for files.

## Data Layers

The space on disks is divided into pages (usually in sizes of 4KB). The main memory is also divided into pages of the same size.

We first look at an overview of how data lays out:

- Disks (HDD, SSD): OS image, persisted database data, swap area\*
- Physical Memory: The physical pages on the main memory
- Virtual Memory: The memory space from the point of view of a process

\* A quick note for the swap area: If the machine is running low on physical memory, it may evict pages from physical memory to the swap area. It needs to keep track of the locations of the swapped pages and the context associated with it.

## Physical Memory

This should be straightforward.

## Virtual Memory

The OS maintains the illusion that each process wholly owns a contiguous area of memory, called **virtual memory**. Each process is free to use its virtual memory space without having to worry about the existence of other processes.

The OS maintains the mapping of pages between the physical memory and the virtual memory of each process. This information is kept in a **page table**. The **page directory** indexes all the page tables for different processes.

The virtual memory space can exceed the physical memory capacity. Not every virtual page has a corresponding physical page. The OS will **delay the allocation** of the physical page until the virtual page is actually accessed. If the number of virtual pages being accessed exceeds the number of physical pages, the system will be running low on memory and will start evicting some pages to the **swap area** on disks. This will significantly affect the system performance so it's unrecommended to have a working set larger than the size of the physical memory.

The virtual memory space is all a process can see. What data is present there?

- **Process stack**: function calls, stack frames
- **Process Heap**:
  - Data buffers for files
    - These buffers are populated from kernel-level page caches (More on page cache later)
    - There could be two copies of the same data.
  - Memory-mapped (mmap) IO devices
    - With `mmap`, a memory region in the user space is mapped to a file. You don't need a separate user buffer for it. You write data to it and the OS will handle the page for you. This is an alternative to `read` and `write` systems calls but it has its own semantics and complications.

### Virtual → Physical translation

The OS consults the page directory and page table to translate the virtual address of a process into a physical address. The page directory provides an index into the page tables.

**How does** a **page directory** reduce **the overall memory overhead?**

- The idea is that if the page table is not used, it can be offloaded from memory.
- The overall page table can be broken down into smaller ones and indexed by the page directory.
- Given a virtual address, the OS consults the page directory to locate the page table containing that address and only needs to load that particular page table into the memory.

There is a hardware unit within the CPU called **TLB** (translation look-aside buffer) which is dedicated to caching and speeding up the translation.

## Page Cache

How does a process read/write files? There are actually a lot of things involved in the OS because the OS itself caches the pages of files and keeps them in kernel-level buffers, called **page caches**.

**Why use page caches?** For performance reasons.

- The OS may be able to prefetch the required pages, hiding the latency.
- The OS can delay and batch the write-back of pages, reducing the number of disk IOs.

From a process's point of view, all it does is invoke the system calls: `open`, `read`, `write`.

Let's say a process is reading an existing file and extending it

- `open`: The process provides a file path to the OS. The OS needs to follow the path and locate the inode of the file. It also needs to check the permissions.
- `read`: The process passes a data buffer to the OS and it's the job of the OS to fill it up with data. The OS reads from disks and puts the data onto the page caches. It then copies the data from the page cache to the user-space data buffer.
- `write`: The process writes to a new location to extend the file. The OS, or the file system part of it, needs to
  - Find an unallocated page
  - Appends that to the metadata of the file
  - Allocates a page cache to store the data being written
  - Marks the page as dirty.

The data is not immediately written to disk with the `write` system call. When to actually flush the data is determined by the OS. The `fsync` system call can make that happen synchronously.
