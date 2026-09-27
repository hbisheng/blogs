---
title: "Computer startup process"
date: "2022-10-04"
weight: 20
---

With a [mental model of program execution]({{< relref "/computer-architecture/program-exec-mental-model.md" >}}) in mind, what happens when you turn on your computer?

1. Read-only memory (ROM) on the mainboard is always available for read once the power is on.
2. Once power is plugged in, CPU starts executing instructions from ROM. The first program is BIOS.
3. BIOS performs **Power On Self Test (POST)** which checks that necessary hardware devices are present.
4. BIOS finds the **boot drive** (the hard disk) and starts the **bootloader** program.
5. The **bootloader** loads the **operating system** image (its programs and data) into memory.
6. The operating system takes control.

Now let's dive into the [operating system]({{< relref "/operating-system/operating-system-overview.md" >}})—the first piece of system software.
