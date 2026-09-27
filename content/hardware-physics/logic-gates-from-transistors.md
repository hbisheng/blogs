---
title: "Building logic gates from transistors"
date: "2023-01-13"
weight: 20
---

**TL;DR** Transistors manipulate voltage levels to build logic gates.

---

Transistors can be used to build logic gates.

## Logic Gate

What's a **logic gate** in the first place? Not worrying about the official definition, I think of a logic gate as a device that takes several input signals and produces one output signal. They are the logical building blocks of computers.

The inputs and outputs of logic gates are binary signals that can be represented as 0 and 1. In reality, a low voltage is used to represent 0 and a high voltage is used to represent 1.

Take an **AND gate** as an example, it only outputs 1 when its two inputs are both 1.

|  |  |  |
| --- | --- | --- |
| A | B | Output |
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

## Building logic gates from transistors

As discussed in the previous post, a transistor is a voltage-controlled switch. Now let's consider how to use those switches to build logic gates. We'll be using two kinds of transistors:

1. enhancement-mode NMOS
   - High gate voltage = ON
   - No/low gate voltage = OFF
2. depletion-mode PMOS
   - High gate voltage = OFF
   - No/low gate voltage = ON

### NOT gate implementation

Let's say we want to implement a NOT gate.

One thing that's obvious is that we need a separate **voltage source** because we need to output 1 when the input is 0.

When your input is 0,

- You want the output connected to the voltage source (ON)
- You want the output disconnected from the ground (OFF)

When your input is 1

- You want the output disconnected from the voltage source (OFF)
- You want the output connected to the ground (ON)

This can be achieved by

- putting a PMOS between the output and the voltage
- putting an NMOS between the output and the ground
- connecting the input to their gates

### PUN and PDN

The network between the output and the voltage is usually called a **pull-up network (PUN)** while the network between the output and the ground is called a **pull-down network (PDN)**.

In a logic gate, the PUN and the PDN are complementary; they always return opposite results (otherwise the voltage will be connected to the ground).

### Series and parallel connection

What happens when we connect two NMOS in series? This combination is only ON when both NMOSs are ON. This is basically an AND. So two NMOSs connected in series is what you need as the PUN of an **AND gate**.

Similarly, if you connect two NMOSs in parallel, the combination is ON as long as one of the NMOSs is ON. You get the PUN of an **OR gate**.

What about the PDN? It needs to be the opposite of the PUN.

For the AND gate, this means the state is only OFF when both inputs are 1. Equivalently, the state is ON when one of the inputs is 0. That basically means connecting two PMOS in parallel.

### AND gate

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| A | B | Output | PUN (pull up network) | PDN (pull down network) |
| 0 | 0 | 0 | OFF | ON |
| 0 | 1 | 0 | OFF | ON |
| 1 | 0 | 0 | OFF | ON |
| 1 | 1 | 1 | ON | OFF |
|  |  |  | series NMOS | parallel PMOS |

Similarly, the PDN of an OR gate is two PMOSs connected in series.

### OR gate

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| A | B | Output | PUN (pull up network) | PDN (pull down network) |
| 0 | 0 | 0 | OFF | ON |
| 0 | 1 | 1 | ON | OFF |
| 1 | 0 | 1 | ON | OFF |
| 1 | 1 | 1 | ON | OFF |
|  |  |  | parallel NMOS | series PMOS |

In fact, given any number of inputs and one output, you can implement the PUN and PDN of any logical functions you want. You'll first get the logical expressions and do the following transformations

- A → connect A to an NMOS
- not A → connect A to a PMOS
- A and B → series connection
- A or B → parallel connection

## References

https://www.geeksforgeeks.org/cmos-logic-gate/

https://www.electronics-tutorials.ws/category/logic/page/2
