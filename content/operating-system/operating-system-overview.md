---
title: "Operational System Overview"
date: "2022-10-04"
weight: 10
---

**TL;DR** OS is a **system program** that provides a curated environment for user programs to run.

---

Think about all the physical aspects of running a program in reality, everything that is not written in the user program is handled by the OS.

User programs can invoke system calls and the OS will take care of it. User programs do not worry about when they can execute; it's all controlled by the OS. A user program only sees a virtual memory space and the OS performs the translation to physical addresses. To take full advantage of the limited amount of memory, the OS also offloads some memory (in the unit of *pages*) to disks when it's not active.

- **Providing kernel service**: Handling system calls, exceptions, timer/device interrupts: IDT and GDT. Each request is associated with a well-defined routine.
- **Process management**: A process is a program in execution. There is a context around it. Each process has its address space.
- **Scheduling**: A scheduler that determines which process should run. Context switches happen when one process is suspended and another is resumed.
- **Memory management**: Virtual memory addresses are translated into physical addresses. The translation is done by looking up multi-level page tables. Translation Lookaside Buffer (TLB) is used to cache that translation. Each process has its own virtual memory space and thus its own page tables.
- **On-demand paging:** The page available to be accessed by a process may not be present physically in memory. The actual allocation is done until the last possible moment. When a process accesses the page and it isn't present, an exception is raised so that the kernel can allocate a new page frame and set up the mapping in page tables.
- **Page Cache:** Page cache is for reading/writing files. Direct I/O can be used to avoid the extra data copy in "disk -> kernel buffer -> user buffer" as it directly transfers the data from disk to user buffer.
- **I/O management**
  - Networking Socket
  - File system

### OS bootstrap

What does it do first, chronically? Set up provisional IDT and GDT. Switch from real mode to protected mode.
