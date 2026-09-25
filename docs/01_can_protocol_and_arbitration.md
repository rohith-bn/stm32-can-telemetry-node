# 01. CAN Protocol Fundamentals, Physical Layer & Arbitration

* **Topic**: Controller Area Network (ISO 11898-1/2) Physical Principles
* **Author**: Rohith B Narasimhamurthy
* **Context**: Learning notes and technical rationale for `stm32-can-telemetry-node`

---

## 1. Why I Studied This

Coming from general embedded development (I2C, SPI, UART), CAN bus requires a paradigm shift. In SPI or UART, you transmit point-to-point between chips. In CAN:
* There are no chip-select pins or slave addresses.
* Communication is broadcast-based: every node on the bus sees every message.
* Messages are tagged with an Identifier that represents the content and priority, not the recipient.

To build an automotive telemetry node, I needed to understand what happens physically on the copper wire before attempting to configure registers on an STM32.

---

## 2. The Physical Layer: Differential Signaling (ISO 11898-2)

Automotive environments are hostile: motor switching, spark plugs, high currents, and alternator noise introduce massive electromagnetic interference (EMI). 

Single-ended signals (like standard 3.3V UART) easily pick up noise spikes that flip logic levels. CAN solves this using **differential signaling** over a twisted pair (`CAN_H` and `CAN_L`):

```
Bus State: Recessive (Logic 1 - Bus Idle)
CAN_H = 2.5V
CAN_L = 2.5V
Differential (Vdiff = CAN_H - CAN_L) = 0.0V

Bus State: Dominant (Logic 0 - Active Transmission)
CAN_H = 3.5V
CAN_L = 1.5V
Differential (Vdiff = CAN_H - CAN_L) = +2.0V
```

### Why Noise Rejection Works
When common-mode noise hits the twisted pair (e.g. +5V noise spike), both `CAN_H` and `CAN_L` rise by +5V simultaneously:
$$V_{\text{diff}} = (3.5 + 5.0) - (1.5 + 5.0) = 8.5 - 6.5 = +2.0\,\text{V}$$
The differential receiver subtracts the two lines, completely canceling the noise.

### Bus Termination (120 Ohm)
The bus lines act as high-frequency transmission lines. Without proper impedance matching, signal edges reflect off the cable ends, corrupting bits.
* A standard CAN bus must have exactly two 120-Ohm termination resistors, one at each physical end of the cable.
* The equivalent DC resistance across the bus between `CAN_H` and `CAN_L` when powered off should be approximately:
$$R_{\text{term}} = \frac{120 \times 120}{120 + 120} = 60\,\Omega$$

---

## 3. Bit-Wise Non-Destructive Arbitration

In Ethernet (CSMA/CD), if two devices transmit simultaneously, a collision occurs; both abort, wait a random backoff time, and retry. This nondeterministic delay is unacceptable for automotive braking, steering, or engine control.

CAN uses **CSMA/CR (Carrier Sense Multiple Access with Collision Resolution)** or non-destructive bitwise arbitration:

1. **Dominant bit (Logic 0)** actively drives current through the bus.
2. **Recessive bit (Logic 1)** passively lets the bus float to 2.5V.
3. Therefore: **Dominant always overwrites Recessive** ($0 > 1$).

### How Nodes Arbitrate
* Before transmitting, a node listens to ensure the bus is idle.
* If two nodes begin transmitting at the exact same instant, they transmit their CAN Identifiers bit-by-bit (MSB first).
* At the same time they drive the bus, **nodes read back the actual bus voltage**.
* If Node A transmits a Recessive `1`, but reads back a Dominant `0` (because Node B transmitted a Dominant `0`), Node A knows a higher-priority message is on the bus.
* Node A immediately ceases transmission and turns into a receiver. Node B never even notices a conflict occurred.

```
Node A (ID 0x100 = 001 0000 0000 binary):
Bit 10: 0 (Dominant)  -> Bus is 0 -> OK
Bit 9:  0 (Dominant)  -> Bus is 0 -> OK
Bit 8:  1 (Recessive) -> Bus is 0 (overwritten by Node B!) -> Node A loses arbitration!

Node B (ID 0x050 = 000 0101 0000 binary):
Bit 10: 0 (Dominant)  -> Bus is 0 -> OK
Bit 9:  0 (Dominant)  -> Bus is 0 -> OK
Bit 8:  0 (Dominant)  -> Wins bus! Node B continues unaffected.
```

### Application to our Telemetry Node
This explains why our frame priorities were designed as follows:
* `0x050` (`NODE_COMMAND`): Lowest numerical ID $\rightarrow$ Highest priority on the bus. Commands from the master ECU can interrupt any telemetry stream immediately.
* `0x100` (`NODE_HEARTBEAT`): Medium-high priority. Vitality of the node must be confirmed regularly.
* `0x200` (`IMU_TELEMETRY`): Medium priority high-rate sensor stream.
* `0x300` (`POWER_DIAGNOSTICS`): Lowest priority telemetry data.

---

## 4. CAN 2.0B Frame Structure

A complete standard data frame consists of:
1. **SOF (Start of Frame)**: 1 dominant bit to synchronize all node clocks.
2. **Arbitration Field**: 11-bit Identifier + RTR bit (0 for data frame).
3. **Control Field**: 
   - IDE bit (0 for standard 11-bit).
   - r0 (reserved dominant bit).
   - DLC (4 bits): payload length (0 to 8 bytes).
4. **Data Field**: 0 to 8 bytes of application payload.
5. **CRC Field**: 15-bit CRC calculation + 1 recessive CRC delimiter bit.
6. **ACK Field**: 
   - ACK slot (transmitter sends recessive 1; any node that received a valid CRC pulls it dominant 0).
   - ACK delimiter (1 recessive bit).
7. **EOF (End of Frame)**: 7 consecutive recessive bits.

---

## 5. Summary and Architectural Takeaways

1. **Message-Centric, Not Node-Centric**: Nodes do not have MAC or IP addresses. The ID defines the content of the frame.
2. **Prioritization by ID**: The lower the binary ID, the higher the bus priority. Critical supervisory frames must have lower IDs than bulk sensor telemetry.
3. **Hard Ceiling on Data Size**: Standard CAN 2.0B frames carry a maximum of 8 data bytes. Every byte must be utilized efficiently through deterministic bit-packing (which leads into our DBC design).
