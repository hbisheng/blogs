---
title: "File system"
date: "2023-02-20"
weight: 50
---

**TL;DR** A filesystem organizes the information on disks to support reading and writing a large number of files.

Information that needs to be persisted is stored on the disk. You want to store a lot of data in it, which requires you to organize the information well. In disks, the space is divided into pages. The interface provided by the hardware disks can be simplified as two APIs:

1. `byte[] read(pageNum long)`
2. `boolean write(pageNum long, byte[] data)`

You need to build a system that stores and retrieves a large amount of data with these APIs. We are talking about a file system, which is part of the operating system.

## Filesystem

What's a file system? It's a tree structure where there are two types of nodes: **directory** and **file**.

- The root is a directory.
- A directory may contain a number of directories and files.
- Files are terminal nodes. Files are containers of information/bytes.

I'm just going to put down a theoretic design of how things work below. It may be far from the real thing.

### Page allocation

Firstly, we need to keep track of is which pages are **allocated** and which pages are **free**.

We could use a **binary bit tree** to represent that information.

- A 16TB disk has 4G pages to manage, which will takes 500MB space in total for the binary bit tree.
  - 32 page lookups in the worse case.
  - This information can be looked up to defragment and stuff.

### Directory and file

We will reserve a few pages to store system-level information.

The root directory can be stored at a specific, well-known page.

**For a directory type, what exactly is stored there?**

- A fixed-size component that stores the creation time, permissions, etc.
- A dynamic-size linkedlist that stores the children of the directory:
  - Each child points to the physical location (page number and offset) of the node.

For a file type, it is similar

- A fixed-size component.
- A dynamic list that contains all the pages for this file and their sizes. You can seek to a particular offset by traversing this list.

The above probably describes what an **inode** is. The inode of a file contains a file allocation table (FAT) and indicates where the content of the files are stored on disks.

### APIs

1. `open(filepath)`: open a file so that it is ready for reading or writing.
2. `read()`: Read from a certain position of a file
3. `write()`: Write to a certain position of a file

These APIs are exposed as system calls by the kernel. We will look closer into what happen exactly for these system calls after we introduce the concept of page caches.

## NFS

There's something called a NFS (network filesystem) that faces vastly different challenges. That warrants a separate post.
