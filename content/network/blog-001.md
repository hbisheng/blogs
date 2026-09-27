---
title: "Network: Socket and the network stack"
date: "2022-10-01"
weight: 10
---

*Q: I have two programs running on two different computers. How do they talk to each other through the network?*

**TL;DR** Socket is an abstraction provided by the operating system for network communication.

---

Each of the two **programs** is running as a **process** on the computer, managed by the **operating system** (OS).

A process can use **system calls** to request services from the OS. One of those services is **sending/receiving** **data** over the network.

#### Socket

The OS provides an abstraction called **socket** to interact with its network interface. As a metaphor, sockets can be considered as communication channels.

Before we talk about how sockets can be used by user programs, let's first look at the two common roles that a process may have in a networking scenario—a **client** or a **server**.

A client proactively **establishes** a network connection while a server passively **waits** for a network connection. This is similar to a phone call where we have a caller and a callee.

As the first step of doing any network communication, both the client and the server need to **create a socket** (via a system call) and **bind** the socket to a certain network address (via another system call). After that, a server will wait for new connections by **listen**ing on the sockets. If there is an incoming connection, it would **accept** the connection and starts **sending/receiving data** over that connection. In comparison, a client simply use the initialized socket to **connect** to a remote address.

Each of above-mentioned socket operations corresponds to a system call. For example, the OS provides the following system calls: `socket`, `bind`, `connect`, `listen`, `accept`, `close`, `recv/send`.  You can tell by their name what each system call does.

#### Network Stack

Now let's try to **follow a piece of data** that's sent over the network.

Suppose that process A wants to send process B **a string** “hello world”. Process A has that string variable stored somewhere in **memory**. Process A first uses system calls to create a socket, bind it to a certain address, and connect to a remote socket address bound by B. A socket address is a pair of IP address and port.

Now, process A issues a system call to send the data over the connection. We will take a look at how that system call is handled by the OS.

The kernel (which is part of the OS) implements a **network protocol stack** that has several layers. A **protocol** means there are assumptions/expectations on both sides for it to function.

Here are 5 common network layers:

1. **Application Layer**: This is about a user program interacting with the the kernel network interface.
2. **Transport Layer**: How data packets are sent to a destination port which identifies a particular process. Common protocols are **TCP** or **UDP**.
3. **Network Layer**: How packets are routed to the correct address via intermediate routers (The **IP** protocol)
4. **Data Link Layer**: How bits are constructed as data frames/packets. Making sure that correct bits are transmitted and reconstructed.
5. **Physical Layer**: How bits are transmitted. The physics that move bits down the wires.

The data to be sent starts as a string variable at the application layer and moves all the way down to the physical layer. At each layer, the network stack **wraps the data** with necessary **headers/checksums** but does not care much about the rest of the content. At the bottom layer, the kernel sends the wrapped data to the **network interface card (NIC)**. After that the data travels through the wire to the computer where process B is running.

On the side of process B, the kernel reads data from NIC, **unwraps** the data through the same network stacks, and forwards the message up to the application layer. The kernel copies the data to the buffer that the application expects. If process B is waiting for data to arrives, it is in a sleeping state waiting for signals. The kernel will then wake it up, after which process B continues its logic and do whatever it wants (printing the data for example).

#### A Go programmer's point of view

<https://www.linode.com/docs/guides/developing-udp-and-tcp-clients-and-servers-in-go/> contains some great examples of writing TCP client/server in Go. Basic function calls:

```
conn, err := net.Dial("udp", "[2001:db8::1]:domain")
_, err = conn.Write("hello world")
```

As an application programmer, you usually only care about the application layer and your business logic. You only need to worry about what data you want to send/receive; you don't need to be constantly thinking about the network stacks except for performance tuning. In my work so far, I have not had the need to understand much about the network stacks other than tuning some TCP buffer size. Things just worked.

But it's always a good thing to dig into more technical details. If you follow the `net` package which is part of the standard library, you will see code that explicitly starts a system call ([code ref](https://cs.opensource.google/go/x/sys/+/fb04ddd9:unix/zsyscall_linux_amd64.go;l=208)).

#### The end

There are many other aspects of networking that I would like to write about but they'll need to be in separate posts:

- TCP handshake. TCP congestion control.
- IP protocol. How the data get relayed by the routers. BGP.
- gRPC: standardized service-to-service communication. protobuf.
- SSL. public key, private key.
- Kafka zero copy. DMA → NIC.
