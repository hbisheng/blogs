---
title: "CPU hyperthreading"
date: "2023-02-10"
weight: 30
---

**TL;DR** Hyperthreading turns a physical core into multiple logical cores and allows starting multiple instructions per cycle. But it can not achieve the same performance as having real physical cores due to shared resources (e.g. CPU caches).

We had a mental model for program execution where the CPU is abstracted as an FSM. This post gets into more detail about the mechanisms in CPUs that optimize the overall performance, especially how hyperthreading works. This [article](http://www.lac.inpe.br/~stephan/CAP-372/MultiThreading_Stokes.pdf) already did an amazing job of explaining the concepts. Much understanding of the topic was derived from that article.

## Pipelining

Recall that a CPU has multiple stages for running an instruction: IF, ID, EXE, MEM, WB.

Naively, the CPU can run one instruction at a time. Assuming that each stage takes one cycle, one instruction can be done with 5 CPU cycles.

|  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- |
| **CPU cycle** | **IF** | **ID** | **EXE** | **MEM** | **WB** |
| 1 | Instruction 1 |  |  |  |  |
| 2 |  | Instruction 1 |  |  |  |
| 3 |  |  | Instruction 1 |  |  |
| 4 |  |  |  | Instruction 1 |  |
| 5 |  |  |  |  | Instruction 1 |

These stages can be pipelined so that the hardware of different stages are better utilized.

|  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- |
| **CPU cycle** | **IF** | **ID** | **EXE** | **MEM** | **WB** |
| 1 | Instruction 1 |  |  |  |  |
| 2 | Instruction 2 | Instruction 1 |  |  |  |
| 3 | Instruction 3 | Instruction 2 | Instruction 1 |  |  |
| 4 | Instruction 4 | Instruction 3 | Instruction 2 | Instruction 1 |  |
| 5 | Instruction 5 | Instruction 4 | Instruction 3 | Instruction 2 | Instruction 1 |
| … |  |  |  |  |  |

CPU uses pipelining to maximize throughput—the number of instructions completed per unit of time. With pipelining, the CPU can now complete one instruction per CPU cycle.

A counterintuitive fact is, the more pipeline stages the CPU has, the higher the clock frequency can be. The clock frequency is bounded by the slowest stage; if a slow pipeline can be broken down into multiple smaller pipelines, the clock can run faster.

## Single-threaded workload

There are different models for handling single-threaded workloads.

1. **In-order execution + one instruction per cycle**
   - This is the simplest model discussed in the previous section.
2. **In-order execution + two instructions per cycle**
   - If a CPU can start multiple instructions per cycle, it's called a **superscalar CPU**.
   - Without changing the order of executions, it's actually hard to pair two consecutive instructions. So the performance improvement is not significant. Note that some say this is not really “in-order” anymore.
3. **Out-of-order execution + one instruction per cycle**
   - Some instructions are independent. CPU can execute instructions out of order as long as it maintains the illusion that they are executed in order.
   - Out-of-order execution can help reduce pipeline stalls.
     - The pipeline is stalled if there's a data hazard (e.g. one instruction is waiting for the result of another).
     - Even when only one instruction can be started per cycle, adjusting the order can be helpful.
       - For example, a later instruction can use MEM first if an earlier instruction cannot use it yet.
4. **Out-of-order execution + two instructions per cycle**
   - This requires a superscalar CPU.
   - ILP (instruction-level parallelism) is fully leveraged.

## Hardware Multithreading

Hardware multithreading can turn a **physical core** into multiple **logical cores** (a.k.a. **hardware threads**). The hardware must maintain the illusion that the thread is solely occupying the hardware resources.

A physical core nowadays can typically support 2 hardware threads. From the OS point of view, the two logical cores can be considered **totally independent** and can execute different threads of executions.

The concept of hardware multithreading must be distinguished from multiprocessing in the software layer.

The following two sections discuss two mechanisms of hardware multithreading.

### Super-threading / Temporal Multithreading / Interleaved Multithreading

In super-threading, the core can switch the execution to a different thread if one thread is stalled (e.g. due to RAM access). The switch is done by reloading the context (e.g. registers).

If a core only has one pipeline (meaning no extra HW resources), super-threading is the only possible way to support hardware multithreading.

### SMT (Simultaneous Multithreading) / Hyperthreading

Hyperthreading needs a superscalar CPU that can initiate multiple instructions per cycle. The instructions can come from different threads of executions. With a superscalar CPU, more than one thread can be executed in a pipeline stage (e.g. fetch instructions for multiple threads in a cycle). To achieve this, some hardware resources—e.g. the program counter and registers—need to be duplicated, and scheduling logic needs to be added. But many components are still shared, like CPU caches and TLB.

### Hardware multithreading performance

By **sharing hardware resources**, hyperthreading can improve the overall utilization of hardware resources; it is good for computations that can be parallel. It is meant to improve the **overall throughput** but not necessarily benefit a particular thread. From a single thread's point of view, it can definitely run faster if it doesn't need to share the core with another thread.

In practice, hyperthreading does not double the performance of a system. But running two threads in parallel should be faster than running the same two threads in series.

Due to the very same reason that hardware resources are shared, the context switches and the resulting cache/TLB misses can lead to punishing performances. For certain workloads, hyperthreading can actually make things worse.

## Reference

http://www.lac.inpe.br/~stephan/CAP-372/MultiThreading\_Stokes.pdf

More resources to read in the future:

- https://www.lighterra.com/papers/modernmicroprocessors/
