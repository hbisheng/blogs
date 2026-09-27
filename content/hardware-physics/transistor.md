---
title: "The physics of a transistor"
date: "2023-01-12"
weight: 10
---

The transistor is one of the marvelous inventions of human beings and yet they are so common in our lives. This post attempts to uncover some of the physical makeup and characteristics of a transistor.

**TL; DR** Transistor is like a voltage-controlled switch.

## The fundamentals

A transistor is like a voltage-controlled switch. How is that functionality achieved? How do human beings make such devices? Let's start with the most fundamental component—silicon.

### **Silicon**

Silicon is a **semiconductor**. Each silicon element has 4 electrons on its outer shell. It needs 8 electrons to be stable. When they are bound together (e.g. in silicon crystalline), each silicon element get 8 electrons by sharing electrons, forming a stable structure. There are no extra electrons or holes. Pure silicon doesn't have extra electrons so its conductivity is low.

### **N-crystal**

If we dope silicon with phosphorus, each element having 5 electrons in its outer shell, in addition to the original structure, we get **excess electrons** that can move around. These electrons act as charge carriers.

### **P-crystal**

If we dope silicon with aluminum or boron, each element having 3 electrons at its outer shell, in addition to the original structure, we get **excess holes** of electrons. These holes can also act as charge carriers.

### **P-N junction**

Interesting things happen when you form a P-N junction by combining the two crystals mentioned above.

1. Free electrons in the N-crystal **diffuse** to the surface of the P-crystal and fill the holes there.
2. The displacement of electrons creates a **potential** that repels further diffusion.
3. When everything stabilizes, a potential of around 0.7V is formed. The electrical field starts from N to P. There are no more free electrons at the junction, forming a depletion zone for charge carriers.

By applying an **external voltage**, this potential can be **overcome** or **enhanced**

- Connect P → +, and N → -, with a voltage larger than 0.7V, this puts the PN junction in **forward bias**. The potential is overcome and electrons can pass through, forming a current.
- Connect P → -, and N → +, this puts the PN junction in **reverse bias**. Electrons cannot pass through.

You know what, this behavior is exactly the functionality of a **diode**! Electric current can flow through one direction but not the other (not considering the breakdown scenario of course).

## **From PN junctions to transistors**

There are two kinds of transistors

1. Bipolar junction transistor (**BJT**)
2. Metal–oxide–semiconductor field-effect transistor (**MOSFET**).

MOSFET is more common nowadays. It's faster to switch because it relies on the field effect. It consumes less power and consumes less space. But let's look at both of them.

### Bipolar junction transistor (**BJT**)

A BJT is like a sandwich where you wrap a P-crystal with two N-crystals or an N-crystal with two P-crystal. You get either an **NPN transistor** or a **PNP transistor**.

A BJT has three pins: **collector**, **base**, and **emitter**. The base is lightly doped; the collector is moderately doped; the emitter is heavily doped. The emitter is usually connected to the ground and there are voltage sources at the base and the collector.

A BJT can be used as a **switch** or an **amplifier**.

Based on different voltage conditions, an NPN transistor has different operating regions/modes

1. **Cutoff region** (**open switch**)
   - Condition: V\_base < 0.6V
     - base-emitter junction: not forward bias.
   - The transistor acts like an open switch between the collector and the emitter.
   - I\_base = I\_collector = I\_emitter = 0A (ignoring leakage between collector and emitter)
2. **Saturation region (closed switch)**
   - Condition: V\_base ≥ 0.6V, V\_base > V\_collector\_emitter
     - base\_emitter junction: forward biased. base\_collector junction: forward biased.
   - The transistor acts like a closed switch between the collector and the emitter.
3. **Active region** (**amplifier**)
   - Condition: V\_base ≥ 0.6V, V\_base < V\_collector\_emitter
     - base\_emitter junction: forward biased. base\_collector junction: reverse biased.
   - The transistor acts like an amplifier.
     - I\_collector = I\_base \* beta.
     - I\_base and I\_collector can be increased by increasing V\_base.
     - V\_collector\_emitter doesn't change I\_collector.

### Metal–oxide–semiconductor field-effect transistor (**MOSFET**)

In my opinion, a MOSFET is closer to the metaphor of a voltage-controlled switch.

You have a **gate** in the middle and **drain** and **source** on both sides.

- NMOS: N-crystal on both sides.
- PMOS: P-crystal on both sides.
- (CMOS: Contemporary)

The gate is insulated with a metal layer; it used **field-affect** to control the conductivity.

Take an NMOS as an example, when a gate voltage is applied, electrons are attracted and build a channel in the p-crystal, connecting the two n-crystal. It also has several operating regions

1. **Cutoff region**
   - V\_gate = 0V.  The default state. No current.
2. **Saturation region**
   - 0V < V\_gate < threshold. There's some current.
3. **Ohmic/Linear/Triode Region**
   - V\_gate ≥ threshold
   - V\_gate determines the upper bound of I\_source\_drain

The MOSFET can have different default modes when no voltage is applied. The default can be ON (**depletion mode** device) or OFF (**Enhancement mode** device). The described example is an enhancement mode device.

#### The physical construction of a MOSFET

Illustrative images can be found at

<https://www.halbleiter.org/en/fundamentals/construction-of-a-field-effect-transistor/>

The whole process looks like planting teeth lol.

---

**Random BJT notes**

(I spent quite some time trying to figure out details of BJT, but later realized that it's not as common as MOSFET. So I didn't bother digging further. Just dumping the notes here.)

**BJT saturation mode**

- **I\_collector\_saturation** depends on your workload.
- Max **V\_collector\_emitter\_saturation** can be found on the hardware sheet. The smaller the better.
- Max **V\_base\_emitter\_saturation** can be found on the hardware sheet. Around 0.7V.
- **I\_base** can be calculated using the beta on the hardware sheet. You need at least this much. But further increasing V\_base has no effect on the collector current.

**BJT active mode**

Base\_collector is reversed biased but it's not a normal PN junction. The base crystal is already filled with electrons due to the forward biased in the base\_emitter. So the electrons can move forward from the base. The higher collector voltage provides higher attraction.

**BJT application**

If you gradually increase the base voltage

- At first, it's in a cutoff region. V\_collector\_emitter is equal to the voltage source due to no current.
- At around 0.6V base voltage, I\_collector starts to increase and V\_collector\_emitter starts to decrease. (active region)
- At around 0.8V base voltage, I\_collector reaches a maximum, V\_collector\_emitter is the smallest. (saturation region)
- Further increasing the base voltage has no effect on I\_collector.

## References

<https://www.halbleiter.org/en/fundamentals/>

<https://www.circuitbread.com/tutorials/different-regions-of-bjt-operation>

<https://www.circuitbread.com/tutorials/how-to-use-a-bipolar-junction-transistor-bjt-as-a-switch>

[[Youtube] How does a reverse biased diode work at the molecular level?](https://www.youtube.com/watch?v=C6Ctnl5RYD0)
