---
title: "Operational Management"
date: "2023-07-03"
weight: 30
---

**TL;DR** Use a workflow platform (e.g. Cadence, Amazon Step Functions) to manage your operations.

In the last post, we talk about the convergence loop as a process that performs operations to converge the actual states with the target states. Let's take a closer look at operations. Operations change the state of the system. Operational management is about managing the ways of doing operations on your platform.

## Background

### What's an operation?

What's an operation exactly? An operation can be as simple as hitting an RPC endpoint or sending a TCP request to a database node. It's just some code logic that needs to be run. The challenge is to manage the operations in a **scalable** manner, with proper **rate limiting**, **auditing** (who does what at what time), **observability** (logging/metric), and **failure tolerance**.

More concretely, examples of operations are: upgrading the OS across the fleet, taking down a host/rack for maintenance, and changing the topology of the Redis cluster to make it more failure tolerant.

### **Manual vs automatic operations**

Initially and conceptually, all operations are manual; it involves humans to start all operations. When we are comfortable and confident with the operations, we make the operations start automatically based on certain signals or conditions. The difference between manual and automatic operations is just whether humans are expected to be involved in the loop.

## Different ways of running operations

### Run it as a CLI

The most primitive form of doing an operation is to wrap the logic inside a CLI. This is good enough for simple and quick operations. This is also a good form for read-only operations that check the status of the system.

The downsides:

- No centralized place to visualize/monitor operations. Hard to apply rate limiting.
- Not good for long-running operations.
- No failure tolerance (if the command fails the whole fails).

### Run it in a service

If you put the logic in a service API endpoint, there's a centralized place for you to apply rate limiting or observability.

You can potentially run an operation **that takes hours**.

There's still no failure tolerance. If an operation fails in the middle, it can be tricky to figure out the current state. You would mostly make each step of your job idempotent so that it's safe to retry. Trying to roll back a partially failed operation would most likely be a nightmare.

#### Worker pool

To make the system more **scalable**, you can create a pool of workers that pick up operations to run.

This can be a dedicated job system for managing the running of operations.

- There's a **queue** of pending jobs.
- There's a **scheduler** that assigns operations to workers.
- You can have a database or a stream to manage the queues and connect job producers and workers.
- Potentially, there are multiple kinds of workers, each responsible for a certain kind of operation. The queues are what connect the workers together.

### Run it as a workflow

Workflow is probably the optimal solution for the management of operations. It has the benefits of scalability (distributed running), failure tolerance (auto checkpointing), context linking, and extended running time.

**What's a workflow?** A workflow is a process with multiple steps. Each step is called an activity.

- You would define your operation as a workflow. It's straightforward because you are breaking down your operation into multiple steps in imperative logic.

**What's the syntax of a workflow?** It depends on the workflow platform. For Cadence, you define your workflow in normal Go/Java code (using the Cadence client library), which is amazing. For the Amazon Step function, you describe your workflow in a JSON or YAML file, which is its own domain-specific language.

During a workflow execution, a workflow worker manages the end-to-end execution, and activity workers handle individual activities. The state of the workflow is persisted so that if a workflow worker fails, another one can pick up and continue seamlessly. Ultimately, the execution of a workflow is like the execution of a state machine.

## Appendix (notes to myself)

### Building a job system with a worker pool based on MySQL

A database for storing all the pending jobs.

A scheduler for assigning jobs to workers.

The workers may advertise the kind of jobs they would take.

When a worker finishes a job, it updates the job status in the database.

The scheduler monitors the heartbeats of workers and marks jobs as failed when needed.

### Cadence architecture in a nutshell

Components

- Three client roles: Workflow starter, workflow worker, and activity worker.
- Cadence service: persisting workflow states, exposing APIs to serve the polling requests of the workers.
- Workflows/activities are defined within the workflow/activity worker service. Workflows/activities are reported(exported) to the Cadence service so that they can be registered.

The process of a workflow execution:

- A workflow worker registers a workflow A by calling the Cadence Service API.
- A workflow starter starts a workflow A by calling the Cadence Service API.
- Cadence workflow persists the scheduling of the workflow A.
- The scheduling of the workflow is sent to the workflow worker via a pull/push? model
- The workflow worker starts executing the workflow. All activities in the workflow definitions are intercepted. The workflow worker first encounters the first activity, schedules the activity by calling the Cadence Service API, and waits for the activity to be completed.
- The activity will be received by an activity worker and be processed. The result will be sent back to the Cadence Service via its APIs.
- The activity completion will create a decision task, which will determine the next activity to process for the workflow.
- The workflow worker will receive the decision task. The previously started workflow execution is most likely available. The workflow worker will send the result of the activity to the workflow to unblock its execution. The workflow will be blocked again when it arrives at the next activity. The process mentioned above will go on another cycle. This process repeats until the workflow completes.
