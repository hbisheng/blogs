---
title: "Network protocol: Ethernet"
date: "2022-11-05"
weight: 20
---

This post describes the **Ethernet protocol**, a common protocol that serves as the **physical layer** and the **data link layer** in the **network stack**.

**TL;DR** The Ethernet protocol allows devices connected through a cable to communicate.

### Background

Every computer is installed with a device called a **Network interface card** (NIC). When we say two computers are connected by cables, we mean that their NICs are connected. NICs can be used by the operating system (OS)  to send and receive data.

### **Physical layer**

Naively, we can abstract the physical layer as the capability that A can send B a **stream of raw bits** while there is no guarantee of reliability. Any bit can be lost in the process. There is of course a lot of physics involved to encode, transmit, and decode the information. That's why it's called **the physical layer**.

As an example, the medium in the cable can be **Copper (**electrical signal; voltage level; high and low) and **Fiber (**optical signal; light frequency; on and off), **No medium** (radio wave; electromagnetic signal; phase change).

When the bits travel through the medium, there is a process of modulation (digital→analog) and demodulation (analog→digital).

- Phones have broadband processors to demodulate RF signals, including WIFI and cellular.

Let's look at a concrete example of bit flowing:

- By applying a high or low voltage over copper wire, information of 1 and 0 can be passed through. Both ends first agree on a frequency of information rates. But there is a problem with clock synchronization between the two linked devices.
- Manchester encoding puts less assumption on the accuracy of the clock. It reduces the number of valid states. Only a drop or a rise is considered a valid signal.

You have two NICs that are on two sides of a cable. They interact with the medium to receive and send bits.

The above description is general in the sense that it's not specific to the Ethernet protocol itself, but the Ethernet protocol does define some physical aspects of bit transmission which I'm not too interested in at this point.

### **Data Link Layer**

The data link layer does the encapsulation/decapsulation **between raw bits and higher-level frames.**

- This is important to know the boundary of frames, and where the actual data starts in the bit stream.

The Ethernet protocol has its definition of the frame format, such as preamble, padding, CRC (for data integrity).

- Inter-frame gap (at least 96 bit-time)
- Preamble (56 bits of alternating 1s and 0s). Also useful for checking the clock frequency.
- SFD (start of frame delimiter): 101011
- Destination MAC Address
- Source MAC Address
- Ethernet Type
- Payload is between 46 bytes to 1500 bytes.
- Frame check sequence (4 bytes)

The data link layer sits between the physical layer and the network layer. It has each two sub-layer:

- **Medium Access Control (MAC) layer (**closer to the physical layer)
  - Interact with the physical layer. Decide when to send, handle collisions (more on this below) and retransmissions.
  - A MAC address is hardcoded into NIC as a physical address for identification.
- **Logical Link Control (LLC) Layer (**closer to the network layer)
  - Provide a **multiplexing** mechanism to allow for multiple network protocols.
    - Different network protocols share the same medium and operate at the same time.
    - LLC is the point of convergence and divergence.

The Data Link Layer is typically implemented in the NIC hardware.

#### **Collision handling protocol**:

- A **collision** causes the intended signals to be indecipherable; it can be detected by the voltage level (copper) or the light frequency (fiber). Some said that it cannot be caught in a wireless network.
- **CSMA**: A device first senses the carrier before transmitting the data to ensure no one else is transmitting. One sends and multiple can listen.
- Ethernet uses **CSMA/CD** (Carrier Sense Multiple Access/Collision Detection).  But if it detects a collision, it **terminates the transmission immediately** (this is the collision detection part), sends a jam signal, and backs off for a certain amount of time that involves some randomness.
- In comparison, **CSMA/CA,** mostly used in wireless networks, **waits for a random period of time before sending** (which CSMA/CD doesn't do) if the channel is busy to reduce the likelihood of simultaneously starting to transmit.

#### Topology

Ethernet also describes different ways for the network devices to connect. Two common **topologies** are **bus** and **star**.

- In the **bus topology**, devices are connected through a long cable. Each device can send data to other devices on the cable. But the cable is a shared medium and **collision** can happen.
- In the **star topology**, every device connects to a hub that relays the data.

### Network devices

Theoretically, two computers can be connected through a physical cable and send Ethernet frames using the MAC addresses. But practically, they are usually connected to a switch or a hub.

**Hub**

- A hub just broadcasts everything to every port. It's at layer 1. It's slow and inefficient.

**Switch**

- A switch maintains a mapping between MAC addresses and physical ports.
- A switch knows the MAC addresses when a device sends data through its ports.
- An unmanaged switch is a layer-2 device that is IP agnostic. A managed switch can operate at layer 3.

### More resources

http://users.ece.northwestern.edu/~rberry/ECE333/Lectures/lec5.pdf
