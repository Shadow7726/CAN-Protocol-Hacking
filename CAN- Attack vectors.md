## Attacks on CAN Bus

### 2.1 Bus Flood Attack

A Bus Flood Attack is a simple denial-of-service attack where CAN frames are transmitted as fast as possible to soak up bus bandwidth, causing legitimate frames to be delayed. Parts of the system may fail when frames don’t arrive on time. The success of this attack depends on the mitigations in place:

- **Open Bus**: Transmitting a frame with a CAN ID of 0 will block all other traffic because it is the highest priority frame.
- **Gateway with ID Filtering**: Only lower priority frames can be delayed, while higher priority frames continue undisturbed.

### 2.2 Simple Frame Spoofing

Frame spoofing is an authentication attack where a receiver accepts a fake frame as if it came from a legitimate sender. Methods include:

- **Direct Connection (e.g., via OBD-II port)**: Queue the CAN frame through the drivers in the firmware of the connected device.
- **Hijacked ECU (e.g., infotainment)**: Use the drivers in the device or new drivers installed as part of the hijacking.

#### Issues with Simple Spoofing:

1. Receivers may act on both legitimate and spoofed frames.
2. Arbitration Doom Loop: If two frames with the same ID enter arbitration simultaneously, both controllers will eventually go into the bus-off state after repeated errors, leading to failure.

### 2.3 Adaptive Spoofing

Adaptive spoofing addresses the problems of simple spoofing by:

- **Listening to the Bus**: The attacker queues the spoofed frame immediately after the legitimate frame, overwriting the receiver's buffer with the spoofed data.

#### Timing:

- The attacker must respond within 8μs at 500kbit/sec.

### 2.4 Error Passive Spoofing Attack

This spoofing subverts the CAN protocol by exploiting Error Passive mode. 

#### Stages of the Attack:

1. **Driving the CAN Controller into Error Passive State**: Generate error frames when the targeted ECU sends frames.
2. **Monitoring and Overwriting**: Overwrite the data and CRC fields with spoofed payload once the targeted frame wins arbitration.

#### Requirements:

- Direct control over CAN TX and CAN RX pins.

### 2.5 Wire-Cutting Spoofing Attack

An attacker with physical access can cut wires to partition the bus and spoof frames to one partition by emulating the other. This is commonly used for odometer fraud.

### 2.6 Double Receive Attack

Exploits a protocol feature where a receiver accepts a frame at the second-to-last bit of the EOF field, and the transmitter accepts it at the last bit. An attacker can force a dominant bit at EOF0 to cause double reception.

### 2.7 Bus-off Attack

This attack drives a targeted ECU offline by incrementing its Transmit Error Counter above 255, forcing it into bus-off state.

#### Recovery:

- Some ECUs automatically recover; others enter a fail-safe mode.

### 2.8 Freeze Doom Loop Attack

This low-level attack freezes bus traffic by repeatedly asserting a dominant bit in the IFS field, exploiting a legacy feature of the CAN protocol.

## 3. Intrusion Detection Systems (IDS)

### 3.1 Introduction

The goal of an IDS is to detect attacks and take action. A software-only IDS can collect data for post-incident analysis but generally cannot prevent attacks. IDS with CAN security hardware can mitigate attacks before they cause harm.

### 3.2 Real-Time Traffic Analysis

CAN bus communication patterns are fixed at design time, allowing for timing analysis to detect anomalies.

#### Process:

- Timestamping the arrival of each CAN frame.
- Comparing frame timing to a pre-calculated timing envelope.
- Detecting timing violations and unexpected frame frequencies.

### 3.3 Payload Analysis

An IDS can look inside the payload of CAN frames to detect suspicious behavior, particularly useful for frames in a diagnostic session.

#### Approaches:

- **Centralized IDS**: Verifies diagnostic tool connection independently.
- **Local IDS**: Runs in each ECU to flag spoofing attacks.

### 3.4 Hardware Support

Hardware support enhances IDS capabilities by allowing pre-reception interrupts, generating error frames, and providing metadata for spoofed frame detection.

#### Example Device: Mercury™ IDS

- Generates interrupts for various events (e.g., SOF, end of arbitration).
- Can trigger error frames to destroy spoofed frames.
- Reports errors and timestamps for detailed analysis.

## Summary of Attacks and Mitigations

| Attack Type                    | Description                                                                                  | Mitigation                                |
|-------------------------------|----------------------------------------------------------------------------------------------|-------------------------------------------|
| **Bus Flood Attack**           | Transmit CAN frames rapidly to soak up bandwidth.                                            | Gateway filtering, IDS detection          |
| **Simple Frame Spoofing**      | Get receiver to accept fake frames.                                                          | Adaptive spoofing, IDS timing analysis    |
| **Adaptive Spoofing**          | Queue spoofed frame immediately after legitimate frame.                                      | Tight timing window, IDS real-time check  |
| **Error Passive Spoofing**     | Drive CAN controller into error passive state and overwrite data.                            | Direct pin control, IDS error monitoring  |
| **Wire-Cutting Spoofing**      | Cut wires to partition bus and emulate one partition.                                        | Physical security, IDS payload analysis   |
| **Double Receive Attack**      | Force dominant bit at EOF0 to cause double reception.                                        | Sequence numbers, IDS timing analysis     |
| **Bus-off Attack**             | Increment Transmit Error Counter to force ECU offline.                                       | Error counter monitoring, IDS detection   |
| **Freeze Doom Loop Attack**    | Repeatedly assert dominant bit in IFS field to freeze bus traffic.                           | Error monitoring, IDS overload detection  |

This structured approach highlights the types of CAN bus attacks and the corresponding intrusion detection and mitigation strategies.

# TEST CASE

| **Test Case ID** | **Attack Type**                   | **Description**                                                                                                          | **Expected Outcome** |
|------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------|----------------------|
| **Test Case 1**  | Bus Flood Attack                  | Transmit CAN frames with the highest priority ID (0x000) at maximum rate. Verify detection and appropriate action.        | System detects attack and filters traffic or alerts. |
| **Test Case 2**  | Bus Flood Attack                  | Flood the bus with lower priority frames. Ensure higher priority frames transmit without delay.                           | Higher priority frames are transmitted without delay. |
| **Test Case 3**  | Simple Frame Spoofing             | Connect a malicious device via OBD-II port. Transmit spoofed frames. Verify IDS detects timing anomalies and alerts.      | IDS detects and alerts on timing anomalies. |
| **Test Case 4**  | Simple Frame Spoofing             | Hijack an ECU and transmit spoofed frames. Verify IDS detection and response.                                             | IDS detects spoofed frames and takes action. |
| **Test Case 5**  | Adaptive Spoofing                 | Transmit spoofed frames immediately after legitimate frames. Verify IDS detects based on timing analysis.                 | IDS detects spoofed frames via timing analysis. |
| **Test Case 6**  | Error Passive Spoofing Attack     | Control CAN TX and RX pins of an ECU, drive it into Error Passive state, and overwrite data. Verify IDS detection.         | IDS detects spoofed frames via error and timing monitoring. |
| **Test Case 7**  | Wire-Cutting Spoofing Attack      | Cut CAN bus wires to partition the bus and transmit spoofed frames. Verify IDS detection through payload analysis.         | IDS detects spoofed frames through payload analysis. |
| **Test Case 8**  | Double Receive Attack             | Force a dominant bit at EOF0 field of CAN frame. Verify IDS detection via timing analysis and sequence checks.             | IDS detects double reception events. |
| **Test Case 9**  | Bus-off Attack                    | Increment Transmit Error Counter of an ECU above 255 by transmitting error frames. Verify IDS detection and ECU recovery.  | IDS detects attack, and ECU recovers appropriately. |
| **Test Case 10** | Freeze Doom Loop Attack           | Repeatedly assert a dominant bit in the IFS field to freeze traffic. Verify IDS detects overload and takes action.         | IDS detects and responds to freeze attack. |
| **Test Case 11** | General IDS Functionality         | Simulate normal and abnormal CAN bus scenarios. Verify IDS classifies traffic and raises alerts for malicious activities.  | IDS correctly classifies traffic and alerts on anomalies. |
| **Test Case 12** | General IDS Functionality         | Evaluate IDS performance and resource utilization under various traffic loads and attack scenarios.                        | IDS operates effectively under different conditions. |

These test cases will help validate the security measures and ensure the robustness of the IDS in detecting and mitigating various CAN bus attacks.
