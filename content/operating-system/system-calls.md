---
title: "System Calls"
date: "2023-02-17"
weight: 40
---

**TL;DR** User programs issue system calls to use the services provided by the operating system. Such services include interacting with the hardware devices.

The operating system is the interface between a user program and the hardware devices. The OS provides services to the user programs. The services are provided through **system calls**. To use hardware resources, users programs invoke system calls.

These are examples where systems calls are used:

- Wait for keyboard inputs
- Print messages on the monitor
- Send out a network packet.

The kernel provides code libraries or APIs for user programs to invoke system calls.

**What happens exactly when a system call is invoked?**

- Those libraries eventually trigger a **software interrupt**.
- There's a context switch from the user space to the kernel space. The kernel will then take control.
- There's a system call handler for the kernel to handle different kinds of system calls. The kernel takes action on behalf of the user programs.
- Once the operation is complete, the kernel passes the control back to the user program and the user program resumes execution.

**How do we typically launch a program?**

- We start by running bash commands or clicking some icons.
- The `shell` program handles the user commands and starts a child process via a `fork` system call.
- The child process inherits quite a lot of attributes, including memory allocation, file descriptors, environment variables, etc.
- The parent and child process both proceed from the same code location. But what follows is a conditional statement that separates the two processes. The parent process moves on but the child process issues a `exec` system call.
- With the `exec` system call, the operating system finds the executable, loads the program into memory, and initiates the program states (memory, stack, heap).
- Now the child process jumps to the entry point of the program and starts running!
