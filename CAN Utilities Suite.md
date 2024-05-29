
# The CAN Utilities Suite

With our CAN device up and running, let’s take a high-level look at the can-utils suite. These tools are essential for working with CAN networks and will be explored in greater detail as we use them throughout this guide.

## CAN Utilities Overview

### asc2log
Parses ASCII CAN dumps into a standard SocketCAN logfile format.

Example:
```
0.002367 1 390x Rx d 8 17 00 14 00 C0 00 08 00
```

### bcmserver
Jan-Niklas Meier’s proof-of-concept (PoC) broadcast manager server that listens on port 28600 by default. It handles repetitive CAN messages.

Example:
```
vcan1 A 1 0 123 8 11 22 33 44 55 66 77 88
```

### canbusload
Determines which ID is most responsible for bus traffic and displays a bar graph of bandwidth offenders.

Usage:
```
canbusload interface@bitrate
```
You can specify multiple interfaces.

### can-calc-bit-timing
Calculates the bit rate and appropriate register values for each CAN chipset supported by the kernel.

### candump
Dumps CAN packets, supports filtering, and logging.

### canfdtest
Performs send and receive tests over two CAN buses.

### cangen
Generates CAN packets and transmits them at set intervals. It can also generate random packets.

### cangw
Manages gateways between different CAN buses, with the ability to filter and modify packets before forwarding.

### canlogserver
Listens on port 28700 (default) for CAN packets and logs them in standard format to stdout.

### canplayer
Replays packets saved in the standard SocketCAN “compact” format.

### cansend
Sends a single CAN frame to the network.

### cansniffer
Interactive sniffer that groups packets by ID and highlights changed bytes.

### isotpdump
Dumps ISO-TP CAN packets.

### isotprecv
Receives ISO-TP CAN packets and outputs to stdout.

### isotpsend
Sends ISO-TP CAN packets piped in from stdin.

### isotpserver
Implements TCP/IP bridging to ISO-TP and accepts data packets in the format `1122334455667788`.

### isotpsniffer
Interactive sniffer designed for ISO-TP packets, similar to cansniffer.

### isotptun
Creates a network tunnel over the CAN network.

### log2asc
Converts from standard compact format to ASCII format.

Example:
```
0.002367 1 390x Rx d 8 17 00 14 00 C0 00 08 00
```

### log2long
Converts from standard compact format to a user-readable format.

### slcan_attach
Command line tool for serial-line CAN devices.

### slcand
Daemon for handling serial-line CAN devices.

### slcanpty
Creates a Linux pseudoterminal interface (PTY) to communicate with a serial-based CAN interface.

## Example Commands

### Setting Up CAN Bus Speed
Set the bit rate and bring up the CAN device:
```bash
$ sudo ip link set can0 type can bitrate 500000
$ sudo ip link set up can0
```

### Troubleshooting CAN Interface
Reset the CAN interface if you encounter issues:
```bash
$ sudo ip link set canX type can restart-ms 100
$ sudo ip link set canX type can restart
```

### Initializing Serial CAN Devices
Configure serial CAN devices and initialize the serial hardware:
```bash
$ slcand -o -s6 -t hw -S 3000000 /dev/ttyUSB0
$ ip link set up slcan0
```
Options for `slcand`:
- `-o`: Opens the device
- `-s6`: Sets the CAN bus baud rate and speed
- `-t hw`: Specifies the serial flow control (HW or SW)
- `-S 3000000`: Sets the serial baud rate
- `/dev/ttyUSB0`: Your USB FTDI device

### Baud Rates
| Number | Baud Rate |
|--------|-----------|
| 0      | 10Kbps    |
| 1      | 20Kbps    |
| 2      | 50Kbps    |
| 3      | 100Kbps   |
| 4      | 125Kbps   |
| 5      | 250Kbps   |
| 6      | 500Kbps   |
| 7      | 800Kbps   |
| 8      | 1Mbps     |

