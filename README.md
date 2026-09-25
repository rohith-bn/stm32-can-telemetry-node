# STM32 CAN Telemetry Node: Applied Automotive System Design

A hands-on, test-driven exploration of automotive CAN 2.0B, Vector DBC serialization, and STM32 bxCAN controller architecture.

* **Author**: Rohith B Narasimhamurthy
* **Development Methodology**: Co-developed with AI (Google Antigravity) as an engineering pair-programmer and learning accelerator.
* **Target Focus**: German Automotive Embedded Systems (ISO 11898-1/2, Vector DBC, MISRA-C:2012 principles, ISO 26262-6 unit verification).

---

## 1. Project Background and Motivation

I am an embedded software developer working toward a career in the German automotive sector. My core background is in embedded firmware and microcontroller systems, but automotive-specific networking—specifically the Controller Area Network (CAN) bus and Vector tooling—was not part of my original curriculum.

In the automotive industry, CAN is not just a peripheral you turn on; it is a mission-critical fieldbus governed by strict timing quanta, hardware arbitration, identifier mask filtering, and standardized DBC database serialization.

Rather than relying on abstract textbook reading or running pre-packaged vendor examples, I created this project to build a complete, production-structured CAN telemetry node from first principles.

### The Objective
* Understand how physical differential signals (`CAN_H` / `CAN_L`) translate into dominant and recessive bus arbitration.
* Master the calculation of nominal bit times, baud rate prescalers, and sample point positioning on microcontroller clock trees.
* Understand how automotive signal matrices are specified in Vector `.dbc` databases and implement deterministic C encoding/decoding (scaling factors, signed offsets, saturation bounds, and endian packing).
* Implement hardware acceptance filters using 32-bit identifier mask modes to eliminate unnecessary CPU interrupts.
* Practice test-driven embedded development (TDD) using a mock Hardware Abstraction Layer (HAL) to achieve 100% host verification before touching physical silicon.

---

## 2. Engineering Methodology: AI-Assisted Progressive Learning

This project was built using an AI-augmented engineering methodology. I do not hide the use of AI; I treat modern LLMs as a tireless senior technical peer, Socratic tutor, and pair-programming assistant.

The workflow followed a disciplined, progressive structure:

```
[Phase 1: Concept Inquiry]
  Identify knowledge gaps in CAN specification (e.g., bit-timing quanta, bus-off recovery, DBC offsets).
  Use AI to dissect datasheets (STM32 RM0090, Bosch CAN 2.0B, ISO 11898).

[Phase 2: Architectural Specification]
  Manually define requirements, message IDs, payload structures, and error states.
  Formulate the Vector DBC database (tools/dbc/telemetry_node.dbc).

[Phase 3: Contract Definition (HAL)]
  Design decoupled C interfaces (can_frame_t, hal_can_t vtable) before any driver logic is written.
  Enforce zero vendor lock-in (no STM32Cube headers inside core application or DBC code).

[Phase 4: Test-Driven Verification (TDD)]
  Write host unit tests and in-memory mock CAN peripheral (tests/host_mocks/).
  Define expected behavior for boundary saturation, Little-Endian bit-packing, and state transitions.

[Phase 5: Implementation & Review]
  Implement signal conversion and state machine logic with AI assistance.
  Rigorously review code line-by-line for MISRA-C compliance:
    - Zero dynamic heap allocation (malloc/free).
    - Fixed-width integer types (stdint.h).
    - Explicit defensive bounds clamping.

[Phase 6: Verification & CI/CD]
  Execute unit test suites natively under GCC (-Wall -Wextra -Wpedantic).
  Configure automated cloud verification via GitHub Actions.
```

Every commit in this repository represents a deliberate step in this learning journey. Detailed architectural decisions and technical learnings are documented in the `docs/` folder.

---

## 3. System Architecture

```
+-------------------------------------------------------------------+
|                     Physical Layer (ISO 11898-2)                  |
|          CAN_H (2.5V - 3.5V) <=======> CAN_L (1.5V - 2.5V)        |
|                    120 Ohm Split Bus Termination                  |
+---------------------------------+---------------------------------+
                                  |
+---------------------------------v---------------------------------+
|                    CAN Transceiver Hardware                       |
|           (SN65HVD230 / TJA1050 3.3V/5V Level Shifter)            |
+---------------------------------+---------------------------------+
                                  | TX / RX (PA12 / PA11)
+---------------------------------v---------------------------------+
|               STM32 bxCAN Controller (ISO 11898-1)                |
|  - 3 Transmit Mailboxes (Priority by ID or Chronological)         |
|  - Dual 3-Stage Receive FIFOs (FIFO 0 / FIFO 1)                   |
|  - 14 Filter Banks in 32-bit Identifier Mask Configuration        |
+---------------------------------+---------------------------------+
                                  |
+---------------------------------v---------------------------------+
|            Board Support Package (bsp_stm32_can.c)                |
|  - 500 kbps Baud Rate Calculation (APB1 = 42 MHz, BRP = 6)       |
|  - Sample Point: 85.7% (BS1 = 11 Tq, BS2 = 2 Tq, SJW = 1 Tq)      |
|  - Register-level Mailbox Transmission and FIFO Reception         |
+---------------------------------+---------------------------------+
                                  |
+---------------------------------v---------------------------------+
|           Hardware Abstraction Layer (hal_can.h)                  |
|  - hal_can_t Vtable Contract: init, transmit, receive, filter     |
|  - Decouples Telemetry FSM from Physical Silicon                  |
+---------------------------------+---------------------------------+
                                  |
+---------------------------------v---------------------------------+
|            CAN Signal Serialization Engine (can_dbc.c)            |
|  - Endian-Safe Bit Packing (Little-Endian / Intel Standard)       |
|  - Mathematical Scaling Factors, Signed Offsets & Saturation      |
+---------------------------------+---------------------------------+
                                  |
+---------------------------------v---------------------------------+
|             Telemetry State Machine (can_telemetry.c)             |
|  - Cyclic Frame Scheduler: 10 Hz Heartbeat, 50 Hz IMU, 5 Hz Power |
|  - Diagnostic State Machine: INIT, PRE_OP, OPERATIONAL, FAULT     |
|  - Autonomous Bus-Off Recovery Monitoring (TEC/REC Evaluation)    |
+-------------------------------------------------------------------+
```

---

## 4. CAN Message Matrix & Vector DBC Specification

The telemetry frame matrix is defined in `tools/dbc/telemetry_node.dbc`:

| CAN ID | Message Name | Cyclic Rate | DLC | Signals & Engineering Units | Conversion Formula |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **0x100** | `NODE_HEARTBEAT` | 10 Hz (100 ms) | 8 B | - Node ID (0x42)<br/>- State (INIT, PRE_OP, OP, FAULT)<br/>- Error Bitmask<br/>- Rolling Uptime (ms) | Direct raw integers |
| **0x200** | `IMU_TELEMETRY` | 50 Hz (20 ms) | 8 B | - Roll Angle: [-180.00, +180.00] deg<br/>- Pitch Angle: [-90.00, +90.00] deg<br/>- Yaw Rate: [-500.0, +500.0] dps<br/>- Accel Magnitude: [0.0, 16.0] g<br/>- Rolling Alive Counter [0, 255] | `Roll = Raw * 0.01`<br/>`Pitch = Raw * 0.01`<br/>`Yaw = Raw * 0.1`<br/>`Accel = Raw * 0.1` |
| **0x300** | `POWER_DIAGNOSTICS` | 5 Hz (200 ms) | 6 B | - Bus Voltage: [0, 36000] mV<br/>- Current Draw: [0, 5000] mA<br/>- Core Die Temp: [-40, +125] deg C<br/>- Bus Warning Flag | `Voltage = Raw * 1`<br/>`Current = Raw * 1`<br/>`Temp = (Raw * 1) - 40` |
| **0x050** | `NODE_COMMAND` | Inbound / Event | 8 B | - Command ID (RESET, PRE_OP, OP)<br/>- 32-bit Command Parameter | Inbound supervisory control |

---

## 5. Repository Directory Layout

```text
stm32-can-telemetry-node/
|-- docs/                        # Architectural records and progressive learning notes
|   |-- 01_can_protocol_and_arbitration.md
|   |-- 02_bit_timing_and_filter_calculations.md
|   `-- 03_dbc_signal_serialization_design.md
|-- bsp/                         # Board Support Package (STM32 bxCAN registers)
|   |-- include/bsp_stm32_can.h
|   `-- src/bsp_stm32_can.c
|-- hal/                         # Hardware Abstraction Layer
|   `-- include/hal_can.h
|-- include/                     # Public contracts and API declarations
|   |-- can_frame.h
|   |-- can_dbc.h
|   `-- can_telemetry.h
|-- src/                         # Core implementation
|   |-- can_dbc.c
|   |-- can_telemetry.c
|   `-- main.c                   # Simulation demo entry point
|-- tests/                       # Host-based unit testing suite
|   |-- host_mocks/              # In-memory virtual CAN peripheral and queues
|   |   |-- include/mock_hal_can.h
|   |   `-- src/mock_hal_can.c
|   |-- test_can_dbc/            # Unit tests for bit packing, scaling & saturation
|   |-- test_telemetry/          # Unit tests for state machine & scheduling
|   `-- run_tests.ps1            # Test execution script (MSYS2 GCC)
|-- tools/
|   `-- dbc/telemetry_node.dbc   # Authentic Vector DBC database
|-- .github/workflows/ci.yml     # Automated CI/CD pipeline
|-- .gitignore
|-- CMakeLists.txt
|-- LICENSE
`-- README.md
```

---

## 6. Host Unit Testing and Verification

A central requirement of this project was test-driven verification. Every serialization function, boundary condition, and state transition was tested without requiring physical hardware:

```bash
# Execute full host test suite via PowerShell runner
powershell -ExecutionPolicy Bypass -File ./tests/run_tests.ps1

# Alternative: Direct GCC command
gcc -Wall -Wextra -Wpedantic -std=c11 -Iinclude -Ihal/include -Itests/host_mocks/include \
    src/can_dbc.c tests/test_can_dbc/test_can_dbc.c -o test_can_dbc && ./test_can_dbc
```

### Verified Test Results (MSYS2 GCC 15.2.0, -Wall -Wextra -Wpedantic):
* **CAN DBC Test Suite**: 42 passed, 0 failed. Verifies Little-Endian packing, negative two's complement, scaling precision, offset calculation, and saturation clamping.
* **Telemetry State Machine Test Suite**: 27 passed, 0 failed. Verifies cyclic rate timing, supervisory state changes, low-voltage/over-temp safety flags, and bus-off recovery.
* **Total**: 69/69 passing tests, 0 warnings.

---

## 7. Key Learnings and Takeaways

Through the development of this node, I gained hands-on competence in areas critical to automotive embedded systems:
1. **Sample Point Sensitivity**: A CAN node cannot simply choose arbitrary bit-time dividers. If the sample point is not set between 75% and 87.5% (CiA standard), clock drift between nodes will corrupt message arbitration.
2. **Deterministic Bit-Packing**: Automotive signals rarely align neatly with byte boundaries. Implementing raw bit-shifting and masking manually built a deep appreciation for the structure of CAN payload frames.
3. **The Power of Mock HALs**: Isolating the bus interface behind function pointers allowed rapid iteration and automated testing on my PC, catching edge-case bugs before hardware integration.

---

## 8. License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
