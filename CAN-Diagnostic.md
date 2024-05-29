
# Brute-Forcing Diagnostic Modes

Each manufacturer has its own proprietary modes and PIDs, which you can usually get by digging through “acquired” dealer software or by using tools or brute force. The easiest way to brute force is to use an open source tool called the CaringCaribou (CC), available at [CaringCaribou on GitHub](https://github.com/CaringCaribou/caringcaribou).

CaringCaribou consists of a collection of Python modules designed to work with SocketCAN. One such module is a DCM module that deals specifically with discovering diagnostic services.

## Getting Started with CaringCaribou

To get started with CaringCaribou, create an RC file in your home directory, `~/.canrc`.

```ini
[default]
interface = socketcan_ctypes
channel = can0
```

Set your channel to that of your SocketCAN device. Now, to discover what diagnostics your vehicle supports, run the following:

```bash
$ ./cc.py dcm discovery
```

This will send the tester-present code to every arbitration ID. Once the tool sees a valid response (`0x40+service`) or an error (`0x7f`), it’ll print the arbitration ID and the reply ID.

### Example Discovery Session

Here is an example discovery session using CaringCaribou:

```
-------------------
CARING CARIBOU v0.1
-------------------
Loaded module 'dcm'
Starting diagnostics service discovery
Sending diagnostics Tester Present to 0x0244
Found diagnostics at arbitration ID 0x0244, reply at 0x0644
```

We see that there’s a diagnostic service responding to `0x0244`. Great! Next, we probe the different services on `0x0244`:

```bash
$ ./cc.py dcm services 0x0244 0x0644
```

```
-------------------
CARING CARIBOU v0.1
-------------------
Loaded module 'dcm'
Starting DCM service discovery
Probing service 0xff (16 found)
Done!
Supported service 0x00: Unknown service
Supported service 0x10: DIAGNOSTIC_SESSION_CONTROL
Supported service 0x1a: Unknown service
Supported service 0x00: Unknown service
Supported service 0x23: READ_MEMORY_BY_ADDRESS
Supported service 0x27: SECURITY_ACCESS
Supported service 0x00: Unknown service
Supported service 0x34: REQUEST_DOWNLOAD
Supported service 0x3b: Unknown service
Supported service 0x00: Unknown service
Supported service 0x00: Unknown service
Supported service 0x00: Unknown service
Supported service 0xa5: Unknown service
Supported service 0xa9: Unknown service
Supported service 0xaa: Unknown service
Supported service 0xae: Unknown service
```

Notice that the output lists several duplicate services for service `0x00`. This is often caused by an error response for something that’s not a UDS service. For instance, the requests below `0x0A` are legacy modes that don’t respond to the official UDS protocol.

> **NOTE**  
> As of this writing, CaringCaribou is in its early stages of development, and your results may vary. The current version available doesn’t account for older modes and parses the response incorrectly, which is why you see several services with ID `0x00`. For now, just ignore those services; they’re false positives. CaringCaribou’s discovery option stops at the first arbitration ID that responds to a diagnostic session control (DSC) request. Restart the scan from where it left off using the `-min` option, as follows:
>
> ```bash
> $ ./cc.py dcm discovery -min 0x245
> ```
>
> In our example, the scan will also stop scanning a bit later at this more common diagnostic ID:
>
> ```
> Found diagnostics at arbitration ID 0x07df, reply at 0x07e8
> ```

## Keeping a Vehicle in a Diagnostic State

When doing certain types of diagnostic operations, it’s important to keep the vehicle in a diagnostic state because it’ll be less likely to be interrupted, thereby allowing you to perform actions that can take several minutes. In order to keep the vehicle in this state, you need to continuously send a packet to let the vehicle know that a diagnostic technician is present.

These simple scripts will keep the car in a diagnostic state that’ll prove useful for flashing ROMs or brute-forcing. The tester present packet keeps the car in a diagnostic state. It works as a heartbeat, so you’ll need to transmit it every one to two seconds, as shown here:

```sh
#!/bin/sh
while :
do
  cansend can0 7df#013e
  sleep 1
done
```

You can do the same things with `cangen`:

```bash
$ cangen -g 1000 -I 7DF -D 013E -L 2 can0
```

> **NOTE**  
> As of this writing, `cangen` doesn’t always work on serial-line CAN devices. One possible workaround is to tell `slcand` to use `canX` style names instead of `slcanX`.
```
