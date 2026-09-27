---
title: "Network protocol: IP"
date: "2022-12-13"
weight: 30
---

**Main Thread:** Logical gates → Registers → Program execution (memory and CPU) → Operating system → Intercommunication through Ethernet → Routing protocols in a network

**TL;DR** The IP protocol routes data packets around the world.

## Routing Algorithms

Now you have several computers connected in a network, how do the data frames get transmitted between them?

If you just have a few devices, they can all connect to the same **switch** and send data to each other via **MAC addresses**.

But if you have a fairly large network, you will need intermediate devices to relay the data. They are called **routers**. Routers have routing table to guide their routing. They route data packets.

Every device on the internet has an identifier which we call its **address**. For now, it can be any string. It just needs to be unique.

How to route the data packets efficiently? It's about choosing the best routes between any two destinations.

Given a certain number of vertexes and edges, this becomes a pure algorithmic problem.

Some assumptions:

- Each node in the network is fully aware of its neighbors.
- Each node can communicate with its neighbor.

### **Distance Vector Routing**

Each node starts with an empty routing table. Each node adds itself to the routing table with a cost of 0.

- Talks to all your neighbors and see their routing table. See their destinations and the number of hops to get to the destinations.
- If it is faster for you to route through them (1 plus their hop counter), update your routing table.
- Send your neighbor (local nodes) your cost to every node (global information).
- Each node updates their cost to every node and iterate until convergence.

Mathematically we should be able to prove that the network will converge into a network with shortest paths.

- The steady state is that hop\_counter(you, dest) = min(hop\_counter(any\_of\_your\_neighbor, dest))+1
- This is called the **Bellman-Ford algorithm**, or **Distance Vector Routing**.
- The problem is that it doesn't handle link failures well. Count to infinity.

This can be summarized as “share global information with local nodes”.

### **Link State routing**

An alternative routing algorithm is **Link State routing**, which uses **Dijkstra's algorithm**.

- Send every node (broadcast) your cost to your neighbor.
- Every node runs the Dijkstra's algorithm on its own.

This can be summarized as “share local information with global nodes”.

## IP Addresses

Earlier we mentioned that each device (with a few exceptions like a network switch) has a unique identifier on a network.

The most common internet protocol uses IPv4 addresses as the identifier.

Each IPv4 address is a 32-bit number. The next-gen IPv6 address is 4 times longer.

- IP address is made up of the Network part (the prefix) and the host part.
- There are public IPs and private IPs. Private IPs are used in an internal network; they are not exposed on the Internet.
- Historically, all IP addresses are divided into three classes: A, B, C.
- Later, CIDR allows for an arbitrary division of an IP address into network and host parts

There are a total of 4 billion IPv4 addresses. There are not enough IPv4 addresses in the world.

All devices in your home actually share the same public IP addresses. This is achieved by **Network Address Translation** (NAT). The router keeps track of a mapping between public addresses and private addresses, ports included.

Network in a limited area (e.g. home or small offices) is called a **local area network (LAN)**.

Next we'll look at how private IP addresses are allocated in a LAN.

### Private IP Allocation in a LAN

Within a LAN, IP addresses are assigned by a **DHCP (Dynamic Host Configuration Protocol)** server.

DHCP is an **application layer** protocol. Each LAN should have exactly one DHCP server (or a relay agent). If there are multiple, they should serve separate IP ranges. Otherwise there'll be IP conflicts.

If there's no DHCP server, a **link-local address** will be used after broadcast checking. Link-local addresses can only be used for communications within the LAN; no requests can be routed out.

If there's a DHCP server, the IP allocation process follows the **DORA principle**:

- **Discovery** (Broadcast): Clients send out requests to find the DHCP server.
  - At the beginning, source IP is set to 0.0.0.0 because there is none. But the source MAC address is specified.
  - Destination IP is set to 255.255.255.255. This indicates a broadcast.
- **Offer** (Unicast): The DHCP server responds with an offer, a candidate IP address.
- **Request** (Broadcast): The client can accept an offer by sending a broadcast request. It is a broadcast to tell other DHCP servers to withdraw their offer.
- **Ack** (Unicast): The DHCP server approves the request.

### **Communication via IP in a LAN**

Now let's suppose that all devices in a LAN are allocated an IP address, how do they talk to each other?

When you have a target IP address at hand, if it's in the same subnet, you need to find its MAC address; otherwise, you need to send it to the default gateway (the router).

How do you tell if the target IP is in the same subnet? By using the **network mask**.

Recall that an IP address is made up of network address and host address. A network mask specifies how many bits (starting from the beginning) of the IP address are the network address.

A network mask defines a subnet. Each device in a subnet shares the same network address. A subnet is a broadcast domain where devices can broadcast information to each other.

With the subnet, you can tell

- If the target in the same subnet.
  - Send a **Address Resolution Protocol (ARP)** broadcast to find the MAC address
  - After that simply use the MAC address for communication.
- If the target is not in the same subnet
  - Send it to the default gateway.
  - You may need an ARP broadcast to find the MAC address of the router as well.

**Routers**

- Routers are the boundary of a subnet (no router is needed within the subnet). Router only matters for outbound traffic.
- Even between routers, ARP is also needed to find the MAC address of the next hop.
- All routers in a subnet know their subnet mask.
- Routers outside of the organization don't need to know about the subnets.

## internet layer

The internet layer sits above different types of layer 2 networks

- Different networks may have different packet formats. But the internet layer connects them together.
- e.g. Ethernet LAN within an office and ATM WAN that connect different offices
- The special, boundary router will do the translation.

## IP Packet

Key information in an IP packet

- Version
- Total Length
- Time to live
- Protocol (For example, ICMP for a ping request)
- Destination Address
- Source Address

## **Useful commands**

- `ipconfig getifaddr en0`
- `curl ifconfig.me`
- `dig www.facebook.com`
- `dig @198.41.0.4 www.facebook.com`
