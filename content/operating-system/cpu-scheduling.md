---
title: "CPU scheduling/throttling"
date: "2023-02-15"
weight: 30
---

**TL;DR** The operating system controls when and how long processes run on the CPUs.

## Background

A **program** is a static sequence of instructions. When it is being executed, it becomes a **process** managed by the operating system. The operating system controls which processes have access to the CPU at a given point in time.

Apart from the construction and destruction states, a process mostly switches between the following states:

- **Runnable**: The process is ready to run and just waiting for its turn on the CPU.
- **Running**: The process is running on the CPU.
- **Blocked**: The process is blocked on some activity e.g. inputs/outputs.

The kernel divides time into slices (a.k.a. time quantum) and schedules the processes onto those time slices. The length of a time quantum is a trade-off between fairness and efficiency:

- Shorter period: More fair scheduling
- Longer period: Better throughput due to less interruption

**Timer interrupt handler:**

- The kernel runs a process by switching the thread of execution to that process. How does the kernel regain control when a process uses up its time? After a preset amount of time is passed, a timer interrupt is sent to the CPU. The CPU pauses the current execution, saves the context (e.g. registers), and switches to the kernel handler. It's at this point that the kernel regains control of the hardware. It can then decide which process should run next.

How does the kernel decide which process to run next? It should give every process a chance to run so none of them is starved. This is the main topic of the post.

## **Scheduling policy**

Linux uses a scheduler called Completely Fair Scheduler (CFS) ([reference](https://manpages.ubuntu.com/manpages/bionic/man7/sched.7.html)).

Threads have priorities (e.g. normal, real-time). Threads with high priority can preempt lower-priority ones.

There's one queue per priority. The scheduler picks the head of the queue, runs it, and puts it back to the tail of the queue.

**Scheduling policies**

- **SCHED\_FIFO**
  - One of the two real-time policies. Execute without time slices.
- **SCHED\_RR (round robin)**
  - One of the two real-time policies. Execute with a time quantum.
- **SCHED\_DEADLINE**
  - Use if the deadline of the task is super important.
- **SCHED\_OTHER**
  - The default policy for processes.
  - Within this priority, each process has an internal dynamic priority called a **nice value**. The value describes how nice the process is to *other processes*, so a negative value means selfishness and high priority to run.
  - It increases when a thread is denied to run and thus increases the likelihood of it being scheduled next time.

## cgroup quota

Cgroups (control groups) can be used to isolate resources and specify CPU quotas. Without such isolation, the processes are free to run on the CPUs and run as fast as the hardware permits.

A cgroup quota is controlled by two factors: `period` (usually 100ms) and `quota`. If a process exceeds its CPU quota within a period, it won't be able to run until the next period.

The length of `period` is a trade-off between throughput and latency.

- Longer period: **Better throughput** because the program can run for a longer time without interruption. But if there's throttling it would have a **bigger impact on latency** because the process won't make progress until the next period.
- Shorter period: **smaller impact on latency** when there's throttling; **worse throughput**.

The value quota only matters as a percentage of the period.

## cpuset

**cpuset** allows a process to be pinned on a certain CPU core. This way, throttling is no longer necessary and the processes are free to use the cores assigned.

- Pros: no throttling; allows more consistent latencies
- Cons: worse burst performance (because your processes won't be able to spill over to other cores)

There're many considerations about the physical layout of the CPU cores when actually enabling cpuset. See this [blog post](https://www.uber.com/blog/avoiding-cpu-throttling-in-a-containerized-environment/) for more.
