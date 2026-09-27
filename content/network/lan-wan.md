---
title: "Network: LAN and WAN"
date: "2022-12-17"
weight: 40
---

**Main Thread:** Logical gates → Registers → Program execution (memory and CPU) → Operating system → Intercommunication through Ethernet → Routing protocols in a network → IP protocol at different scales

**TL;DR** The IP protocol allows different kinds of networks to interconnect.

---

It always amazes me how routers can route a packet from me to any IP address in the world. How is it so intelligent? Later I realized there was a metaphor that would make more sense to me

- Think of it as the postal system. You can send your letter anywhere in the world from your local postal office.
- But your local post office doesn't know the route to your destination. It just needs to know where the next hop is, which could be the regional dispatch center or whatever. The next hop will dispatch the letter further.
- Each postal office or dispatch center is like a router.

We will look at IP routing at different scales of networks in this post.

## Home/Small office

This is a typical **local area network (LAN)** scenario where the network is for a limited space.

Your **internet service provider (ISP)** provides you with internet access. They give you a cable of some sort

- Phone lines (Digital Subscriber Line, or DSL)
- TV cables (Cable Internet)

The ISP sends you WAN analog signals.

There is a device calleda **Modem** that demodulates the ISP WAN signals into Ethernet signals.

- The modem is the interface between **WAN** and **LAN.**WAN signal on one side; LAN signal (Ethernet) on the other.
- Although its name suggests this, I don't think the Modem is the only device that does the modulation and demodulation of signals.
  - In fact, any network device does it.
    - For example, your phone and laptop.
    - The wireless card and built-in antenna are usually called wireless broadband or baseband processors.
  - Signals are demodulated into digital signals when parsed and analyzed
  - Signals are modulated into analog signals of different kinds during transmission at the physical layer.

There is a router that connects to your modem. Your router provides wired or wireless access for your devices.

Intercommunication within a LAN has been discussed in the last post.

## Medium Size Company

In a medium size company, you might have a spacious office on one floor, multiple floors of a building, or multiple buildings that are closed by.

### Subnetting

A single router won't cut it. The network is inefficient, if not infeasible, when all devices broadcast to each other.

The network needs to be divided into multiple parts for easier management. Each piece is called a **subnet**.

A subnet can be created using physical separation or a VLAN.

- A physically separated group of computers plus an outbound router can form a subnet.
- A **VLAN** (layer 2) could customize the broadcast domain for different ports and simulate physical separation.

There will be routers that aggregate the subnets. So in essence, all devices can still form a LAN.

### Signal coverage

To ensure network coverage, you need multiple routers placed at strategic locations in your offices.

**Wireless Access Point (WAP)** is convenient for extending the outreach of routers and providing wireless coverage.

- It has no DHCP service and no firewall. It just relays information. So you don't have another router to manage.
- It's like connecting through cables but through RF.
- It only has an Ethernet port. It cannot connect directly to the Modem.

WAP is sometimes preferred over wifi routers for easier management.

- If you have multiple wifi routers, they form different subnets.
- wifi routers have different configurations. They have DHCP to manage.

**Possible signal flow:**

ISP network → ISP modem → (wifi) router

- → wired laptop
- → WAP  → mobile device
- → wifi router → mobile device
- (→ mobile device)
- Wifi extender
  - Wire extension: WAP can connect to the wifi (router through wires) and act as a wifi extender as well.
  - Wireless extension: A wifi extender is not amplifying the original router signal; it broadcasts its own and has its own name.

## Large Company

You will have a large corporate network to manage when you have a company that spans multiple regions or even continents.

Each office connects to the local ISP but uses a dedicated network backbone that's not shared with the Internet.

It uses the **MLPS** which is fast and reliable but expensive. Think of it as dedicated two-way highway traffic.

**SD-WAN** is becoming more dominant. It provides centralized WAN management by software, leveraging different kinds of hardware to optimize traffic control.
