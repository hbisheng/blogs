---
title: "A mental model of program execution from logical functions"
date: "2022-10-04"
weight: 10
---

This post is about my mental model of program execution based on the building blocks of computer—logical functions.

**TL;DR** Every bit of magic in computer science starts from logical gates.

### Logical function

A function is a many-to-one relation. Each function has an **input** (domain) and an **output** (converse domain).

A **boolean function** is a function with inputs and outputs limited to *binary values*.

Three basic boolean logical functions: AND, OR, NEG.

By connecting the input and output of different functions, we form more complex functions. Out of all functions that can be formed, if the output depend entirely on the inputs, they are called **combinational functions**.

In the physical world, they can be implemented by **AND, OR, NEG gates**. How they are implemented is in the realm of physics. Combination circuits can be assembled based on those gates to achieve combinational functions. For example, an Adder.

### Adder

Now let's look at how a simple logical unit called an Adder can be defined on top of the logical functions.

An adder is defined as a logical function that adds two numbers and produces the result. Like any combinational logic, it has inputs and outputs. Its inputs are two integers within a value range represented by a certain number of bits. The number of bits representing an input/output is the *width* of the input/output. The output is an integer which is the sum of the two input integers. There is a possibility of overflow if the output width is smaller or equal to the input widths.

At each bit position, there are three inputs, A[i], B[i], C[i-1] and two outputs, O[i], C[i]. A combinational function can be drawn from the truth table with 2^3=8 rows.

An adder is achieved by connecting a number of such components at each bit position.

### Combinational Logic vs. Sequential logic

In combinational logic, the outputs depend entirely on the inputs.

In sequential logic, the outputs not only depend on the current inputs but also the previous inputs. The sequence of the inputs matters.

Next we'll to illustrate how a single bit of information is stored in a [D flip-flop](http://localhost:3000/en/computer/organization/d-flip-flop).

### Set-Reset Latch

Two NAND gates can make a set-reset latch.

It's symmetrical with a feedback loop. Whenever the input of one side drops to 0, the output of that side pops to 1. It's an exception when both inputs are 0 though.

|  |  |  |  |
| --- | --- | --- | --- |
| Input\_A | Input\_B | output | output\_negated |
| 0 | 0 | 1 | 1 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 0 | 1 |
| 1 | 1 | maintain the previous state | |

### D flip-flop

A D flip-flop can store one bit of information, with the following inputs/outputs:

- Inputs: Enabled, Data
- Output: Data

When Enabled is 1, the output is set to be the input.

When Enabled is 0, the output remains unchanged; it remains at the input value the last time when Enabled is 1.

|  |  |  |
| --- | --- | --- |
| Data | Enable | Output |
| 0 | 1 | 0 |
| 1 | 1 | 1 |
| 0 | 0 | maintain the previous state |
| 1 | 0 | maintain the previous state |

A D flip-flow can be made from a set-reset latch. You just need to properly construct the inputs to the set-reset latch.

- Input\_A = NAND(Data, Enable)
- Input\_B = NAND(Data\_negated, Enable)

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| Data | Enable | Output | Input\_A | Input\_B |
| 0 | 1 | 0 | 1 | 0 |
| 1 | 1 | 1 | 0 | 1 |
| 0 | 0 | maintain the previous state | 1 | 1 |
| 1 | 0 | maintain the previous state | 1 | 1 |

Its circuit contains a feedback loop. The intermediate inputs of the output contain the output itself. It has to be designed in a way that, when Enabled is 0, the output stays where it is. It has to be self-compatible. A circuit containing a feedback loop does not allow unstable/flapping states.

**Store signal**

If you directly connect the Enabled to a clock, the output will always be set to the input whenever the clock signal is high. But that's not usually what you want to do, you want to change the output only when the data is fully ready. For that, you can use a Store control signal to the clock signal.

**Rising edge D flip flop**

You may only want to set the data when there is a clock-rising edge. That can be achieved by having another D flip flop to remember the Data signal when the clock value is 0. When the clock signal is 1, the latched Data will be sent to the next D flip-flop. The non-intuitive part is, when the clock signal is 1, the Data signal no longer matters; the Data only matters right before the clock signal is set to 1.

### Registers

Registers can be built on top of flip-flops. When Enabled is 1, multiple bits can be stored. The input of a register is its write bus; its output is its read bus.

### Memory

Memory is a digital circuit that can store and retrieve a certain number of bits based on a position. The position is provided by a number of bits, called the address.

Memory can also be assembled with combination circuits (D flip-flop), although in reality it is usually implemented in sequential logic. Set the address and the read/write bit, the data will be stored/retrieved after a few clock cycles.

### FSM (Finite State Machine)

An FSM describes a finite number of states and how the state can transition from one to another depending entirely on the input and the state before the transition. An FSM can be described by a table with a size of (number of states \* number of possible inputs). Each table grid has the next state based on the corresponding combination of state and input.
*How to convert an FSM into a logical circuit?*

- The circuit first needs to store the current state, possibly with a register.
- The state is transitioned whenever there is a rising/falling clock signal.
- The next state is computed by a combinational circuit, with the current state and the FSM input as the inputs. Whenever there is a state transition, the next state replaces the current state.
- The FSM output can depend entirely on the current state or the FSM input as well. It is constructed based on a combinational function.

### CPU (Central Processing Unit)

What is a CPU? An FSM for executing a set of instructions. Its state includes all its registers.

A CPU is an integrated circuit with inputs and outputs. It is built with awareness and expectations of external systems, including memory devices and the operating system. A CPU recognizes a set of instructions. The instructions and data come from memory.

Its pipeline includes several stages:

- IF (Instruction Fetch)
- ID (Instruction Decode)
- EX (Execution): ALU arithmetic based on the registers.
- MEM (Memory Access)
- WR (Write Result)

Each stage usually takes multiple clock cycles. This is especially true when CPU, which has a very high clock frequency, is waiting for external inputs. During the IF stage, when CPU is waiting for memory to respond, it could process other information and thus have multiple internal states. But all those states belong to the IF stage and have one common trigger—the data from memory is available. When that piece of data is ready, CPU reads in the data, stores it in one of its registers, and moves on to the next stage.
