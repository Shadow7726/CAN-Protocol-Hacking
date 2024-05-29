
# SavvyCAN: A Powerful Tool for CAN Bus Communication and Car Hacking

There are various software tools available for monitoring and filtering CAN communication. These tools range from costly and proprietary to free and open-source. The purpose of this article is to help you get started with car hacking on a budget, focusing on affordable and effective tools. While tools like can-utils and Wireshark are effective, SavvyCAN offers additional features that make it a standout choice.

## Introduction to SavvyCAN

SavvyCAN is an open-source CAN bus reverse engineering and capture tool. It is cross-platform, developed using QT and C++. Initially designed to work with EVTV hardware, it now supports any socketCAN compatible device, as well as the Macchina M2 and Teensy 3.x boards. SavvyCAN can capture and send data to multiple buses and CAN capture devices simultaneously.

Visit the [SavvyCAN website](https://www.savvycan.com/) for more information.

## Why Choose SavvyCAN?

SavvyCAN provides a user-friendly GUI that makes it easy to navigate, filter packets, and work with arbitration IDs. For those new to car hacking, this graphical interface is invaluable. For more experienced users, SavvyCAN offers advanced features such as scripting on CAN frames, fuzzing, and several reverse engineering tools bundled together.

## Setting Up CAN Bus Speed

To configure the CAN bus speed, you need to set the bit rate, which determines the speed of the bus. Typical values for high-speed CAN (HS-CAN) are 500Kbps, while lower-speed CAN buses often use 250Kbps or 125Kbps.

```bash
$ sudo ip link set can0 type can bitrate 500000
$ sudo ip link set up can0
```

Once the `can0` device is up, you can use can-utils tools on this interface. Linux uses netlink to communicate between the kernel and user-space tools. You can access netlink options with the following command:

```bash
$ ip link set can0 type can help
```

## Troubleshooting

If you experience odd behavior such as a lack of packet captures or packet errors, the interface may have stopped. For external devices, try unplugging or resetting them. For internal devices, use the following commands to reset:

```bash
$ sudo ip link set canX type can restart-ms 100
$ sudo ip link set canX type can restart
```

## Configuring Serial CAN Devices

External CAN devices usually communicate via serial interfaces. Even USB devices on a vehicle often use a serial interface, typically through an FTDI chip from Future Technology Devices International, Ltd.

The following devices are known to work with SocketCAN:
- Any device supporting the LAWICEL protocol
- CAN232/CANUSB serial adapters ([CAN232](http://www.can232.com/))
- VSCOM USB-to-serial adapter ([VSCOM](http://www.vscom.de/usb-to-can.htm))
- CANtact ([CANtact](http://cantact.io))

### Note

If you are using an Arduino or building your own sniffer, you must implement the LAWICEL protocol (also known as the SLCAN protocol) in your firmware. Refer to the [CANUSB manual](http://www.can232.com/docs/canusb_manual.pdf) and the [SLCAN API documentation](https://github.com/linux-can/canmisc/blob/master/docs/SLCAN-API.pdf) for details.

To use a USB-to-serial adapter, initialize both the serial hardware and the baud rate on the CAN bus:

```bash
$ slcand -o -s6 -t hw -S 3000000 /dev/ttyUSB0
$ ip link set up slcan0
```

The `slcand` daemon translates serial communication to the network driver `slcan0`. The following options can be passed to `slcand`:
- `-o`: Opens the device
- `-s6`: Sets the CAN bus baud rate and speed (see Table 3-1 below)
- `-t hw`: Specifies the serial flow control, either HW (hardware) or SW (software)
- `-S 3000000`: Sets the serial baud, or bit rate, speed
- `/dev/ttyUSB0`: Your USB FTDI device

### Baud Rates

| Number | Baud Rate  |
|--------|------------|
| 0      | 10Kbps     |
| 1      | 20Kbps     |
| 2      | 50Kbps     |
| 3      | 100Kbps    |
| 4      | 125Kbps    |
| 5      | 250Kbps    |
| 6      | 500Kbps    |
| 7      | 800Kbps    |
| 8      | 1Mbps      |

