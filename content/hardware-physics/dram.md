---
title: "DRAM under the hood (transistors and capacitors)"
date: "2023-01-14"
weight: 30
---

**TL;DR** DRAM stores data in billions of memory cells where each cell has a transistor and a capacitor.

In our mental model of program execution, we imagined the memory of a computer to be built from **logical gates**. That's not entirely accurate. It's true that registers are built from D flip flops which are based on logical gates. But those memories are relatively small.

The main memory of a computer is pretty large; it's in the scale of GBs. It's rather expensive, if not infeasible, to build a main memory based on the same mechanism we build registers. In the real world, a **DRAM (Dynamic Random Access Memory)** is used. This post will take a deep dive into it.

[This awesome video](https://www.youtube.com/watch?v=7J7X7aZvMXQ) really explains it all. Most of the technical details in this post came from it.

## DRAM

A mental model for DRAM is no different from a model for memory in general, it's a device that stores a certain number of bits at different addresses that can be read from or written to. Functionally that's all it does. But today we want to appreciate more about its internals, the specifics, the procedures, and the mechanisms.

The main memory of a computer is installed on the motherboard in the form of a **DIMM** (dual in-line memory module). It's also called a **RAM stick**. It contains several RAM chips (usually 8 or 9) and supports 64 bits per read/write.

When we say a DIMM has 16GB of memory, it actually has a hundred billion memory cells (!). There's no way to get around that :) How on earth is a device with such a crazy number of memory cells manufactured? It's just a magnificent masterpiece of human technology and yet it's so common in our lives.

### Memory Cell

Take a **1T1C memory cell** as an example, it has one **transistor** and a **capacitor**.

- A capacitor is relatively simple. It's two conductive surface areas separated by an insulator (The insulator needs to be **dielectric** but I didn't dig into what that entails. I heard that it helps store energy). The capacitor can store some amount of electrons. A high voltage is used to represent 1 and a low voltage is used to represent 0.
- The transistor, as discussed in previous posts, is a voltage-controlled switch. In a memory cell, it controls whether to expose the voltage of the capacitor to the outside world. The gate of the transistor is connected to the **word line**.
- The capacitor and the **bit line** are on the two sides of the transistor (source and drain).

In a DRAM, you have billions of such cells arranged perfectly in rows and columns. Let's get to the numbers.

### Numbers

The numbers here of course depend on specific products; they are only meant to give a sense of the scale.

Each read or write of DIMM operates on 64 bits of data. This is called the **word length**.

A DIMM contains 8 **chips** and operations are performed concurrently. A ninth chip may be used for error correction.

For a 16GB DIMM, there are 237 bits in total. That's about 137 billion cells.

- Each of the 8 **chips** is responsible for 8 bits per read-write. Stores 234 bits.  17 billion cells.
  - Each chip has 32 (25) **banks**, divided into 8 **groups**.
    - Each bank stores 229 bits. That's about 536 million cells.
    - Each bank has 65536 **rows** (216) and 8192 **columns** (213).

How many bits are needed to identify a row?

- 5 bits to choose the bank and 16 bits to choose the row. 21 in total.

What many bits of addresses are needed to address that 8 bits in a chip?

- 231 possibilities. So 31 bits.

These 31 addressing bits are passed twice to the chip, first opening a row (21 bits) and then reading the columns (10 bits).

### Operations

DIMM has pins that are exposed on the outside. Those pins include **address** lines**,** **data** lines**,** and **control** lines.

A **Decoder** is used to convert the row address into the row selection signal where only the target row is 1 while the rest are 0s.

A **multiplexer** is used to select the target columns when the entire row is available to read.

The two components above can be fully covered by combinational logic.

#### Read

For any of the operations to be performed, **a row must be opened first**. What does it mean for a row to be opened? It means the voltages stored in the capacitors in that row are exposed to the outside world.

To account for the fact that the amount of electrons stored in a capacitor is so small, a **sense amplifier** is used to get the data out. Before connecting the capacitors to the bit lines, the sense amplifier puts a 0.5 voltage on the word line. After the connection, the sense amplifier detects the disturbance induced by the capacitors and drives the voltage to either 0V or 1V. This opens a row.

After a row is open, the **read driver** will be able to read the values out.

#### Write

Similar to a read, the row must be opened first. After that, the **write driver** that is much stronger overrides the existing voltages in the opened row, storing new states in the capacitors.

#### Refresh

Voltage in the capacitor can slowly leak and DRAM needs to periodically refresh the memory cells.

Refreshing a row takes about **50ns**. Refreshing all rows takes about **2ms**. A refresh is done every ~**60ms**.

## Reference

https://www.youtube.com/watch?v=7J7X7aZvMXQ
