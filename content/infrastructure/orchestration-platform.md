---
title: "Orchestration Platform"
date: "2023-05-29"
weight: 20
---

An orchestration platform is faced with the following task: You are given a number of (e.g. 10K) physical machines. Your users want you to run their services; they specify certain resource requirements but other than that do not care where the service will run. Now you have to figure things out.

First, it's worth noting two different kinds of services: **stateless** and **stateful** ones.

### Stateless vs Stateful

What's a stateless service?

- A stateless service processes each request independently and solely based on the information that comes with it, without relying on the context of previous requests. A stateless service doesn't store any context or session between requests.
- It's probably wrong to say that each instance doesn't persist any state; each instance surely maintains certain states in memory and potentially on disk. However, the states stored in a stateless service are derived states that can be derived from other sources of truth; those states can also be considered temporary. Instances of a stateless service are interchangeable. If one fails, you can easily replace it with another instance.

On the other hand, a stateful service maintains some context across requests.

- The processing of one request may depend on previous requests. A stateful service remembers its clients, and their sessions, and maintains information or state about each client. There can be states persisted for each individual client session (e.g., to track the state machine of activities for an individual user) or states persisted globally.
- Stateful services are the sources of truth for certain information. You usually need to have multiple replicas to ensure that there's no data loss when one instance goes down. The instances are not interchangeable.
- Database hosts are examples of a stateful service. It maintains the state of data across individual client requests. The correct processing of a GET request depends on the state set by a previous PUT request.

Why bother to distinguish the two? In the context of orchestration platform, stateless services are easier to handle because they are easily interchangeable.

1. Draining case
   - There are many cases where you want to proactively remove/drain an instance, e.g. due to host maintenance or a kernel upgrade. For a stateless service instance, you can do it gracefully by removing the node from discovery, shutting it down and launching another one. But for a stateful service, more steps may be required (e.g. to replicate the data elsewhere) for a more graceful exit. Only after that is the instance safe to drain.
   - The failure case is not that different though. Both stateless and stateful services need to tolerate a sudden loss of a node.
2. Startup case
   - The startup of a stateful service instance may be more complicated as well. For example, to add a new node to a Redis Cluster, in addition to starting the container, several commands need to be issued for the node to join the cluster and take up traffic (slot movement).
   - Each application can be different. The logic needs to be defined when the application is being integrated into the orchestration platform.

### Architecture

This section discusses a potential architecture of an orchestration platform.

#### Components

**Config store:** There is a database or config store that stores the target states of your system. The config describes a hierarchy of objects (e.g. instances, clusters, nodes). It contains general information such as resource requirements for each node and technology-specific settings.

**Scheduler:** The first consumer of your config is a scheduling service. It's a stateless service. It decides which hosts your nodes should run on. The placement information can be persisted somewhere as the source of truth.

**Host agent:** On each host, there is a host agent running. The agent controls what happens on the host. It may issue commands to dockerd or systemd to launch or shut down services, managing their lifecycle. The target service that gets started may start other processes as well. The host agent also collects the actual running states of the services.

#### **Convergence loop**

This is the key control logic that reconciles any difference between the target states and the actual states.

**State machine**

The convergence loop works like a state machine. It checks for issues, performs some operation, waits for some signal, and performs the next operation, until the actual state of the system matches the target state.

**Issues**

Issues can be defined on the actual states when they deviate from expectations. The issues can serve as signals/triggers for the convergence loop.

There could be issues at different independent layers. Examples:

|  |  |  |
| --- | --- | --- |
| **Layers** | **Issues** | **Reconciliation** |
| Container | Container is not up. | Start the container and wait for its health. |
| Application | The MySQL dynamic config variable is different. | Apply the config variable to the process. |

Not all issues are reconciled immediately and automatically. Sometimes, the issues are risky so manual intervention is needed. Sometimes, the issues are too complicated for an automatic operation

**Centralized vs distributed**

The convergence loop can be a centralized process—a service that reads the target states, collects actual states from host agents, and issues commands to each host agent to reconcile the differences.

Alternatively, this logic can happen in a distributed fashion at each host agent; each host agent reads the target state of its host and takes actions to fix the differences. This is more scalable and extensible (the host agent can potentially perform more complicated/long-running actions).

**Universal vs Application-specific**

Some reconciliation logic is universal while others are application-specific.

- **Universal**: For example, container startup and shutdown are universal operations.
- **Application-specific**
  - For example, you may have a cluster of MySQL running on version 5.7. You want to replace them with 8.0 version. You have to write some logic (operator) to handle these “advanced” differences.
  - Or, you want to build a Redis Cluster that's multi-zone failure tolerant. The orchestration platform can try to distribute the nodes onto as many zones as possible, but other than that, you need to build some logic to actually construct the cluster (decide which nodes are leaders/followers).
  - Application-specific reconciliation logic is mostly needed for stateful applications. For a stateless application, there's not much to configure other than the size of the deployment. Even if there is maintenance to a certain host/rack, instances can be taken down and replaced quite easily.

When a new application/service is onboarded to an orchestration platform, especially stateful ones, some integration logic needs to be written to tell the platform how to handle certain state deviations.

### Kubernetes: an example

**Terminology**: A **pod** is the smallest and simplest unit in the Kubernetes object model that you create or deploy. A pod represents a single instance of a running process in a cluster and can contain one or more containers. Containers in the same pod can share resources and dependencies, and communicate with each other.

**Config store:** etcd.

**Scheduler:** Kubernetes scheduler does exactly what we described above. It places pods onto hosts/nodes. The placement info is stored in etcd.

**Host agent:** Kubelet is the host agent that manages the containers through `containerd`. It reports actual state information to the control plane every 10 seconds (configurable).

**Convergence loop:** Kubelet monitors the desired pod states by talking to the Kubernetes API server (which in turn reads from etcd) and ensures that all of the pods assigned to its node are running and healthy. It can restart unhealthy pods locally.

For scalability concerns, the API server can shard its requests of different kinds to different etcd instances. This may not be a common practice though.
