---
title: "Network Protocol: BGP"
date: "2023-01-02"
weight: 60
---

**Main Thread:** Logical gates → Registers → Program execution (memory and CPU) → Operating system → Ethernet → Routing protocols → IP protocol at different scales → TCP → BGP protocol

**TL;DR** BGP is the routing protocol for the Internet. BGP peers across different autonomous systems exchanges full routing paths to destinations.

---

TCP enables us to transmit data packets (relatively) reliably between two endpoints. Usually, after this point, we'll move on to talk about layers above the TCP, e.g. application protocol. But this time we'll go down to the IP layer again and talk about BPG. BGP is an interesting protocol as it doesn't fit cleanly into the OSI layers. It relies on TCP to exchange information but it makes changes to routing tables at the IP layer, which is below TCP.

We'll revisit the concept of a WAN and discuss how BGP connects the biggest WAN—the Internet—together.

### Border Gateway Protocol (BGP)

We've already talked about two routing protocols

1. Distance Vector Routing (Bellman-Ford algorithm)
2. Link State Routing (Dijkstra's algorithm)

BGP is a routing protocol that's classified as **Path Vector Routing**. It is quite similar to Distance Vector Routing where you broadcast global information locally. When communicating routing information with neighbors, in addition to the **distance**, a BGP node sends out the entire **path** to the destinations it knows. It is more expensive to maintain this information, but this allows for more complex routing policies.

Why is this Path Vector Routing needed? In the real world, Link State Routing is can only be used at a small scale because it needs to broadcast to all nodes in the network, which is not scalable. Distance Vector Routing is possible but it's too naive for real-world scenarios where we want to define more flexible or complex routing decisions.

By maintaining the entire paths to the destinations, BGP allows you to choose any routing path as you see fit, considering factors such as latency, number of hops, etc.

### Autonomous System

In the real world, the Internet composes of many **autonomous systems** **(AS)**. Let's examine the concept of an AS

- Each AS is formed by a network of devices.
- Each AS owns some range of IPs, something like 22.91.39.0/24 or 18.5.0.0/16.
- Each AS could be managed by a certain organization, which could be a university, a large company, an ISP, etc.
- Each AS has an identifier of a number.

Within each AS, the routing of packets is managed by the AS itself. Other ASs do not know or care. Routing protocols used inside an AS are known as **Interior Gateway Protocols** (IGPs). These include the two basic routing protocols mentioned earlier: Distance Vector Routing and Link State Routing.

BGP, on the other hand, is one of the **Exterior Gateway Protocols** (EGPs). It is used to exchange routing information between ASs. BGP nodes are edge routers running at the boundary of an AS and they are set up to connect to their BGP peers in other ASs. They know all the routing information inside their AS and they represent the entire AS when exchanging information with their peers.

And the protocol they use to exchange information is TCP. This is why the introduction of BGP is reserved until after TCP :)

### Anycast

Different network devices can announce the same IP address and a request to that IP can be routed to any of those devices. This is called Anycast.

At the BGP level, we can imagine that multiple ASs announce the same IP address. In the routing table, for each IP range, we have a list of possible routes. BGP doesn't care whether the routes lead to the same device (unicast) or different devices (anycast). So anycast plays well with BGP.

Anycast can be used for load-balancing purposes. Your requests to a certain IP may be routed differently to different servers depending on your location.
