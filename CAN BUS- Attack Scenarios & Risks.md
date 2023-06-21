<p align="center">
  <img  width="180" src="ghost.png" />
</p>

<h1 align="center"> <b>Can Hacking</b></h1>
<h3 align="center"><b>Security Issues in CAN BUS- Attack Scenarios & Risks</b></h3> 

### No verification or identification of messages:

- Attackers can send messages without any verification or identification.
- Messages from unauthorized sources can be treated as valid.
- For example, a frame carrying information to stop the engine can be sent to both the engine ECU and the Infotainment system, leading to potential safety hazards.
- Unauthorized software updates can be sent and executed, potentially causing harm to the overall system.

### All devices connected have access to the same medium:

- Every node that has access to the bus can write to and influence all nodes on the bus.
- If an attacker has access to one device on the bus, they can potentially compromise all other devices connected to the same bus.
- Each node connected to the CAN bus can be a potential attacker.

### Using Broadcasting:

- Every node that has access to the bus receives all messages transmitted on the bus.
- An attacker could sniff and record messages, potentially leading to misuse of the recorded information.
- An attacker can flood the bus with messages, causing unexpected behavior in the system. For example, flooding the CAN bus with a message that has an identifier set to 0, which gives it the highest priority.

### Lack of encryption and key protection:

- Most of the time, CAN bus communication is unencrypted to save the overhead of decryption.
- Software updates without any key protection can be sent and executed, posing a risk to the overall system.
- Encrypting CAN frame payloads is not trivial, especially with the limited 8-byte payload size. Using a block cipher to encrypt the payload is challenging.

### Lack of defined security standards:

- There is no defined standard to enhance the security of the CAN bus.
- It is not straightforward to disturb the synchronization of the CAN bus protocol, as synchronization relies on specific transitions during the frame.

### Popular Attack Types:

#### Replay attack: 
Messages can be stored and later broadcasted, even when the message is no longer valid. This is meant to mislead or deceive other nodes in the network. The aim is to recreate and exploit the conditions at the time the original message was sent.
#### Passive eavesdropping attack: 
This refers to monitoring the network to track vehicle movement or communications. Attackers intercept and analyze the messages passing through the network to gather information about vehicles and their communication patterns.

### Additional Attack Vectors:

## Bus Spoofing:
  - Falsify or inject CAN messages onto the bus.
  - Impersonate legitimate ECUs by sending forged messages.
  - Disrupt or manipulate communication between ECUs.
## Bus Flooding:
  - Overwhelm the CAN bus with a high volume of messages.
  - Cause denial-of-service (DoS) conditions by saturating the bus bandwidth.
  - Disrupt or delay legitimate communication between ECUs.
## Bus Injection:
  - Inject malicious commands or data into the CAN bus.
  - Exploit vulnerabilities in ECU firmware or software to gain unauthorized control.
  - Manipulate vehicle or system behavior by injecting false data.
## ECU Firmware Attacks:
  - Identify and exploit vulnerabilities in ECU firmware.
  - Gain unauthorized access to ECUs or modify their behavior.
  - Abuse weak authentication mechanisms or firmware update processes.
## ECU Malware:
  - Inject malicious code or malware into an ECU.
  - Use compromised ECUs to launch attacks on other parts of the system.
 ## ECU Replay Attacks
- Capture legitimate CAN messages and replay them later.
- Exploit weak message authentication or validation mechanisms.
- Manipulate system behavior or deceive other ECUs.
## ECU Denial-of-Service (DoS):
- Overload or crash specific ECUs by sending malicious messages.
- Exploit vulnerabilities in ECU software or firmware.
- Disable critical vehicle or industrial system functions.
## ECU Physical Access Attacks:
- Gain physical access to an ECU for tampering or extraction of sensitive data.
- Extract firmware or cryptographic keys for further analysis or exploitation.
- Manipulate or sabotage ECUs to cause system malfunctions.
## Bus Eavesdropping:
- Monitor and capture CAN bus traffic for analysis.
- Intercept and decode messages exchanged between ECUs.
- Gather information about the system's functionality or potential vulnerabilities.
## Bus Jamming:
- Transmit continuous noise or interference on the CAN bus.
- Disrupt or prevent communication between ECUs.
- Trigger fault conditions or false alarms in the system.
