
# Attacks on CAN Bus

## 2.1 Bus Flood Attack
A Bus Flood Attack is a simple denial-of-service attack where CAN frames are transmitted as fast as possible to consume bus bandwidth. This causes legitimate frames to be delayed, leading to system failures when frames don’t arrive on time. The success of the attack depends on the mitigations in place.

For an open bus, transmitting a frame with a CAN ID of 0 will block all other traffic due to its highest priority. If a gateway only allows certain IDs, only lower priority frames can be delayed, while higher priority frames continue undisturbed. This is why standard OBD-II diagnostic frames have IDs 0x7df and higher, giving them very low priority.

## 2.2 Simple Frame Spoofing
Frame spoofing is an authentication attack where a receiver accepts a fake frame as if it came from a legitimate sender.

- **Direct Connection (e.g., OBD-II port):** Queue the CAN frame through the drivers in the firmware of the connected device.
- **Hijacked ECU (e.g., infotainment):** Use the device’s drivers or new drivers installed during the hijacking.

The challenge with this approach is that legitimate frames are also received, and receivers may act on both the legitimate and spoofed frames. Additionally, if two frames with the same ID enter arbitration simultaneously, a bit error occurs, causing an error frame and leading to a resynchronization and arbitration restart. This creates an "Arbitration Doom Loop," increasing the Transmit Error Counter (TEC) and eventually driving both controllers into the bus-off state within approximately 4ms on a 500kbit/sec CAN bus.

Problems if the legitimate ECU goes bus-off:

- **Permanent Fail-Safe State:** The ECU might treat the failure as a wiring fault and refuse to communicate.
- **Cessation of Frames:** Other ECUs may detect the fault and move into a fail-safe state.

An attack may intentionally push a vehicle into a fail-safe state, resulting in a form of denial-of-service attack.

## 2.3 Adaptive Spoofing
Adaptive spoofing addresses simple spoofing problems by listening to the bus to see when the legitimate frame is sent and then queuing the spoofed frame to avoid clashes. Many designs store received frames in a buffer for a control loop to look at later. If the spoofed frame is sent immediately after the legitimate frame, it can overwrite the buffer, and the receiver will likely act on the spoofed frame.

The attacker must respond to the frame receive interrupt within a very tight timing window to queue a spoofed frame in time for arbitration. This deadline is just 8μs at 500kbit/sec.

## 2.4 Error Passive Spoofing Attack
Simple spoofing can be detected by monitoring the bus for timing anomalies. Detection is complicated by Error Passive spoofing, which exploits the CAN protocol’s Error Passive mode. When a CAN controller is Error Passive, it cannot signal errors properly and relies on other devices to detect the error and resynchronize.

### Stages of the Attack:
1. **Drive the Targeted ECU’s CAN Controller into Error Passive State:** Generate error frames when the ECU sends any CAN frame, increasing the TEC.
2. **Monitor the Bus and Overwrite Frames:** After the legitimate frame wins arbitration, the attacker overwrites the data and CRC fields with a spoofed payload. The legitimate sender detects an error but cannot signal it while error passive, leaving the attacker’s spoofed frame on the bus undetected by traffic analysis.

This attack requires direct control of the CAN TX and CAN RX pins on the transceiver, meaning an OBD-II dongle with an external CAN controller cannot be used.

## 2.5 Wire-Cutting Spoofing Attack
If an attacker can physically access and cut wires on the CAN bus, they can spoof frames to one partition by emulating the other partition. This attack is used to spoof odometer readings, causing the dashboard to show reduced mileage despite the ECU outputting correct values.

## 2.6 Double Receive Attack
The Double Receive Attack exploits a CAN protocol feature where receivers accept a frame at the second-to-last bit of the EOF field, and the transmitter at the last bit. A bit error in the last EOF bit causes the transmitter to signal an error and resend the frame, leading receivers to accept the same frame twice.

This can be detected by including a sequence number in frames, though most systems are not designed to handle this rare problem. An attacker can control the CAN TX and RX as general-purpose I/O to trigger this error deliberately, making a simple state machine in software to emulate part of the CAN protocol.

## 2.7 Bus-Off Attack
A Bus-off Attack targets an ECU to drive it offline, while other ECUs continue operating. This could be part of a wider attack or a simple denial-of-service attack on a fleet of vehicles. The attack targets all frames from the same ECU, forcing its TEC above 255 and the ECU’s CAN controller to go bus-off.

## 2.8 Freeze Doom Loop Attack
The Freeze Doom Loop Attack exploits a legacy CAN protocol feature where a dominant bit in the first bit of the inter-frame space (IFS) signals an overload condition. The attacker asserts a dominant bit on the CAN TX pin at the first IFS bit and repeats this process, effectively freezing the bus traffic for as long as desired without increasing error counters.

