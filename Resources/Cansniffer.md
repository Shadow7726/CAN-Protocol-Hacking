
# Grouping Packets with `cansniffer`

The `cansniffer` command line tool groups packets by arbitration ID and highlights bytes that have changed since the last time the sniffer looked at that ID. For example, the figure below shows the result of running `cansniffer` on the device `slcan0`.

![cansniffer example output](figure-5-2.png)  
*Figure 5-2: cansniffer example output*

You can add the `-c` flag to colorize any changing bytes.

```bash
$ cansniffer -c slcan0
```

The `cansniffer` tool can also remove repeating CAN traffic that isn’t changing, thereby reducing the number of packets you need to watch.

## Filtering the Packets Display

One advantage of `cansniffer` is that you can send it keyboard input to filter results as they’re displayed in the terminal. (Note that you won’t see the commands you enter while `cansniffer` is outputting results.) For example, to see only IDs `301` and `308` as `cansniffer` collects packets, enter this:

```
-000000
+301
+308
```

Entering `-000000` turns off all packets, and entering `+301` and `+308` filters out all except IDs `301` and `308`.

The `-000000` command uses a bitmask, which does a bit-level comparison against the arbitration ID. Any binary value of `1` used in a mask is a bit that has to be true, while a binary value of `0` is a wildcard that can match anything. A bitmask of all `0`s tells `cansniffer` to match any arbitration ID. The minus sign (`-`) in front of the bitmask removes all matching bits, which is every packet.

You can also use a filter and a bitmask with `cansniffer` to grab a range of IDs. For example, the following command adds the IDs from `500` through `5FF` to the display, where `500` is the ID applied to the bitmask of `700` to define the range we’re interested in:

```bash
+500700
```

To display all IDs of `5XX`, you’d use the following binary representation:

| ID  | Binary Representation |
| --- | --------------------- |
| 500 | 101 0000 0000         |
| 700 | 111 0000 0000         |
|     | ------------------    |
|     | 101 XXXX XXXX         |
|     | 5 X X                 |

You could specify `F00` instead of `700`, but because the arbitration ID is made up of only 3 bits, a `7` is all that’s required.

Using `7FF` as a mask is the same as not specifying a bitmask for an ID. For example:

```bash
+3017FF
```

is the same as:

```bash
+301
```
```
