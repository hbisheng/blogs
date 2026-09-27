---
title: "Disks basics (HDD and SSD)"
date: "2023-01-15"
weight: 40
---

**TL;DR** HDD stores data in spinning disks with magnified regions. SSD stores data in charge traps; it writes in pages and erases in blocks.

DRAMs or registers are considered **volatile memory**; the information gets lost when unplugged from power. We need **non-volatile memory** to persist and store data for the long term. The two common kinds of non-volatile memory are HDD and SSD. We'll cover some basics of it.

## HDD (Hard Drive Disk)

[This video](https://www.youtube.com/watch?v=wtdnatmVdIg) illustrated the whole thing pretty well with animations.

The **disk/platter** is made of metal. There could be multiple **platters**. The functional magnetic part is a 120nm thin layer of Cobalt Chromium Tantalum Alloy (Combination of elements Co, Cr, Ta). The material contains **small magnetic domains** that can be manipulated via external magnetic fields.

There's a **head** that flies 15nm above the platter. Current creates a strong magnetic field that forces the magnetic domain to align. The platter **spins** so that the head can reach any location on it.

There are > 500,000 **tracks** (concentric circles) on the platter. The tracks are divided into **sectors**. In each sector, the information is structured in the following way:

- Preamble
- Address
- Data (**4KB**)
- ECC (Error-correcting code)

Information is encoded as **the change** of magnetic field across neighboring magnetic domains. A change signifies 1 and no change means 0.

There's also a **DRAM component** that buffers data and aids the control components.

Recent development focuses on further **increasing the data density**, such as pre-heating the magnetic domains with lasers.

## SSD (Solid State Disk)

Watch [this video](https://www.youtube.com/watch?v=5Mh3o886qpg) for fancy animations.

SSD uses flash memory which is used by phones and tablets as well. There's no mechanical part in it.

In some ways, it's similar to DRAMs because it stores information in **memory cells**. But those memory cells can persist data even without a power supply. Let's look at an individual memory cell.

### A NAND memory cell

The memory cells stores some **quantity of electrons** in **charge trap flash**. They can hold electrons for decades.

A cell can hold **different levels** of electrons and represent different bits of information. More bits of information make data storage **cheaper but slower** to extract the information.

- SLC: 1 bit
- MLC: 2 bits
- TLC: 3 bits
- QLC: 4 bits

In earlier years, floating gate transistors were used to store electrons. That was changed to a charge trap in recent years.

**Operations**

1. Program (1 → 0)
   - The **control gate** is applied with a **large positive voltage**. The electrons cross the barrier via **quantum tunneling** (!) and get trapped. This changes the **threshold voltage** of transistors.
2. Erasure (0 → 1)
   - The control gate is applied with a **large negative voltage** and the electrons are evicted (through tunneling again).
3. Read
   - Get the level of electrons in the trap by **testing the threshold voltage** of the transistor.

### The structure and numbers

Memory cells are stacked vertically to form **strings.** The structure is called a V-NAND (**Vertical**-NAND). It was named NAND because the way the transistors are connected is similar to a NAND gate.

A string can be thought of as a column where you can read one value at a time. By duplicating the strings many times, you get a **block** where each individual row is a **page**. (Sources say that a string has around 100 cells stacked together. Suppose that's the case, with the simple duplication above, a block will only have exactly 100 pages. Since a block can have 256 pages, I would tend to speculate that by duplicating strings you get physical **rows** and multiple rows form a block. This is too much detail/speculation into the physical layout. **It suffices to say that a block contains multiple pages**.)

With a block, you can read one page at a time.

A page is the smallest unit that can be programmed. Common page sizes are 4KB/8KB/16KB.

A block is the smallest unit that can be erased. Erasing data requires a high amount of voltage and it's hard to restrict the impact to an individual page. This has complications that we'll discuss in the next section.

Just to give a sense of the scale for a 2TB SSD:

- Each SSD → 18 NAND chips
- Each chip → 8 dies x 2 planes/die
- Each plane → 4K blocks
- Each block → 256 pages
- Each page → (8192 bytes \* 8 = 65536) memory cells

The block size above is 2MB. Block size can be as high as 8MB. In that case, the page size might be 16KB and there are 512 pages per block.

### **Complications**

Although it's faster than HDD, SSD has its own complications and challenges.

##### **Write amplification**

One major constraint, as we mentioned, is that you can't erase an individual page; you need to erase an entire block. There is actually no way to “overwrite” a page. To “overwrite” a page, you are really reading out content that is still valid, erasing the whole block, and reprogramming the data into it. This causes **write amplification** where you only intend to write 8KB of data but behind the scene, 4MB of data needs to be written.

##### Garbage Collection

So it is usually preferred to write to clean pages rather than “overwriting” an existing page that's no longer in use. Old pages that are no longer needed are **garbage collected** at some point.

##### Sophisticated Controller

It's obvious at this point that SSD requires a much more sophisticated controller to manage all this. It needs to decide where the page of data should go and keeps track of a lookup table for address translation (user address → physical address). The lookup table is stored in a DRAM component, which is used to buffer read/write data as well.

The controller can decide to write to multiple chips at the same time, maximizing the throughput.

##### **Corrosion**

While quantum tunneling sounds amazing, the barrier that gets tunneled by electrons wears out little by little. A flash memory cell can only sustain a limited number of Program/Erase cycles. This limited the lifespan of an SSD device.

##### **Wear leveling**

To extend the lifespan of the device, the controller needs to be smart about where to write the next page to evenly distribute the load.

##### **Read Disturbance**

Due to the NAND structure, to read the value of a single cell, you need to apply a voltage **high enough** for neighboring cells to pass through, causing some disturbance to the neighbors. After some number of such disturbances, the number of electrons trapped in the cells is no longer so reliable. Even for read-only loads, the controller might decide to erase the blocks and move the data elsewhere, causing further write amplification.

## Quick Comparison

|  |  |  |
| --- | --- | --- |
|  | **HDD** | **SSD** |
| **Cost** | $ | $$$ |
| **Latency** | ~5ms | ~0.05 ms |
| **Throughput** | ~100MB/s | ~500MB/s |
| **Lifespan** | 3~5 years on average | 10 years (may be much shorter due to heavy load) |

## References

https://www.youtube.com/watch?v=wtdnatmVdIg

https://www.youtube.com/watch?v=5Mh3o886qpg

https://www.youtube.com/watch?v=r2KaVfSH884

https://www.extremetech.com/extreme/210492-extremetech-explains-how-do-ssds-work
