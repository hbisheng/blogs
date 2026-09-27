---
title: "Network protocol: TCP"
date: "2023-01-01"
weight: 50
---

**Main Thread:** Logical gates → Registers → Program execution (memory and CPU) → Operating system → Intercommunication through Ethernet → Routing protocols in a network → IP protocol at different scales → TCP layer

**TL;DR** TCP is an end-to-end protocol at the transport layer that achieves reliable data transmission over an unreliable network.

---

The IP protocols enable us to send data packets across a wide network. But it's unreliable; data packets can be lost or delivered out of order. We need a layer on top of it to make it more usable from the point of view of a message sender or receiver. That layer is the transport layer where we have the **transmission control protocol** (**TCP**).

TCP is an **end-to-end** protocol, meaning that the protocol is only applicable to the sender or receiver of the message but not the intermediate relayer.

Each of the two **endpoints** (sender and receiver) in a TCP connection is identified as a **pair of IP** addressesandports. **Ports** are numbers that represent logical/virtual places of a computer where information can be exchanged.

A TCP connection in essence is the memory states stored on both ends; it's an agreement. Any TCP packet can be routed to your device but it'll only be recognized and accepted if it follows the TCP protocol and matches your expectation.

### TCP establishment

TCP RFC: <https://www.ietf.org/rfc/rfc793.txt>

The 3-way handshake process

1. The client sends SYN with a (random) client-side sequence number.
2. The server replies ACK/SYN and a new server-side sequence number.
3. The client sends ACK.

After the handshake phase, the client connection is added to the list of pending connections waiting to be accepted by the server. The server will use system calls to accept pending connections.

**Why a three-way handshake?**

- Why not a two-way handshake?
  - **Each way needs an SYN and an ACK**
    1. **Reachability**: After the first two messages, the second party doesn't know for sure whether its message can reach the first party;  The third message (ACK), the confirmation, is thus necessary.
    2. **Sequence number initialization:** The second party needs to confirm its initial sequence number with the first party.
  - **Reduce confusion by duplicated SYNs:** Suppose that the first party intends to establish one connection but sends multiple SYN requests due to network delay. With a third message, the second party can double-check which SYN is valid instead of wasting resources on invalid ones.
    - You might say that the first party can send an RST to invalidate the connections upon receiving an invalid ACK, but the problem is the first party may never receive that ACK because the packet is lost. In that case, you will end up wasting a connection on the server for several minutes before it gets idle timed out.
- Why not four-way?
  - Three is enough to know that the channel is *usable*. Having more handshakes will not change that conclusion.
  - Due to the two generals‘ problem, no matter how many handshakes you have, you can never *guarantee* that the other side has the TCP connection established. A three-way handshake is practical enough.

### **Sequence Number**

- Two random sequence numbers were randomly chosen in the initial handshake.
- ACK from the other side comes with a sequence number that it **expects** next.
- SYN increments the sequence number by one on the destination.
- Each byte of data transmitted also increments the sequence number by 1.
- When the sequence number reaches MAX\_UINT it wraps around to 0.

### **Flow control and congestion control**

**Sliding window protocol**

- The sender maintains a window containing the data that is being sent to the other side but has not yet been ACKed. If the head of the window gets ACKed, the window will move forward. The max window size can be dynamic, depending on flow control and congestion control.

**Flow-control:**

- The sender gets feedback on the available space of the receiver and adjusts its speed (through an ACK).

**Congestion control:** Estimate the sending rate of the network.

- (a) Slow start: Exponential growth until a threshold
- (b) Congestion avoidance: linear growth

**From the sender/receiver's point of view:**

- Sender: It chooses the minimum of the congestion window and the receiver's window and sends only that much data.
- Receiver: TCP keeps the data in its receive buffer until the receiving application reads it from that buffer. If the receiving application can read the data fast enough, a larger receive window can improve performance.

### TCP Connection Termination

**3-way handshake but 4 times close:**

- FIN (The sender has nothing more to say)
- ACK (This allows the receiving party to transmit the remaining data if needed)
- FIN (The receiver has nothing more to say)
- ACK

Both sides need to send a FIN and receive an ACK for it. The ACK and FIN in the middle cannot be combined because there might still be data to be sent.

There is a TIME\_WAIT state after the sender sends the final ACK and the wait time is 2 MSL (Max Segment Lifetime ) which is arbitrarily set to 1 minute or 2 depending on the OS.

- This gives enough time for the receiver to receive the final ACK (this will take one MSL at max) or retry the FIN (this will take two MSLs at max).
- This time gap serves to reduce the confusion where data segments meant for this TCP connection leak into the next TCP connection with the exact time addresses and ports.

### **TCP header format**

- Sequence number
- Acknowledgment number
- Checksum
