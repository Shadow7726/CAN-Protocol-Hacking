
# Reversing the CAN Bus with OpenXC

Depending on your vehicle, one solution to reverse engineering the CAN bus is OpenXC, an open hardware and software standard that translates proprietary CAN protocols into an easy-to-read format. The OpenXC initiative was spearheaded by the Ford Motor Company. As of this writing, OpenXC is supported only by Ford, but it could work with any auto manufacturer that supports it. For information on how to acquire a pre-made dongle, visit [OpenXC](http://openxcplatform.com/).

Ideally, open standards for CAN data such as OpenXC will remove the need for many applications to reverse engineer CAN traffic. If the rest of the automotive industry were to agree on a standard that defines how their vehicles work, it would greatly improve a car owner’s ability to tinker and build on new innovative tools.

## Translating CAN Bus Messages

If a vehicle supports OpenXC, you can plug a vehicle interface (VI) into the CAN bus, and the VI should translate the proprietary CAN messages and send them to your PC so you can read the supported packets without having to reverse them. In theory, OpenXC should allow access to any CAN packet via a standard API. This access could be read-only or allow you to transmit packets. If more auto manufacturers eventually support OpenXC, it could provide third-party tools with more raw access to a vehicle than they would have with standard UDS diagnostic commands.

> **Note:** OpenXC supports Python and Android and includes tools such as `openxc-dump` to display CAN activity.

The fields from OpenXC’s default API are as follows:
- `accelerator_pedal_position`
- `brake_pedal_status`
- `button_event` (typically steering wheel buttons)
- `door_status`
- `engine_speed`
- `fuel_consumed_since_last_restart`
- `fuel_level`
- `headlamp_status`
- `high_beam_status`
- `ignition_status`
- `latitude`
- `longitude`
- `odometer`
- `parking_brake_status`
- `steering_wheel_angle`
- `torque_at_transmission`
- `transmission_gear_position`
- `vehicle_speed`
- `windshield_wiper_status`

Different vehicles may support different signals than the ones listed here or no signals at all.

OpenXC also supports JSON trace output for recording vehicle journeys. JSON provides a common data format that’s easy for most other modern languages to consume, as shown below:

```json
{
  "metadata": {
    "version": "v3.0",
    "vehicle_interface_id": "7ABF",
    "vehicle": {
      "make": "Ford",
      "model": "Mustang",
      "trim": "V6 Premium",
      "year": 2013
    },
    "description": "highway drive to work",
    "driver_name": "TJ Giuli",
    "vehicle_id": "17N1039247929"
  }
}
```

*Listing 5-4: Simple JSON file output*

Notice how the metadata definitions in JSON make it fairly easy for both humans and a programming language to read and interpret. The above JSON listing is a definition file, so an API request would be even smaller. For example, when requesting the field `steering_wheel_angle`, the translated CAN packets would look like this:

```json
{
  "timestamp": 1385133351.285525,
  "name": "steering_wheel_angle",
  "value": 45
}
```

You can interface with OpenXC with OBD like this:

```bash
$ openxc-diag --message-id 0x7df --mode 0x3
```

## Writing to the CAN Bus

If you want to write back to the bus, you might be able to use something like the following line, which writes the steering wheel angle back to the vehicle, but you’ll find that the device will resend only a few messages to the CAN bus.

```bash
$ openxc-control write --name steering_wheel_angle --value 42.0
```

Technically, OpenXC supports raw CAN writes, too, like this:

```bash
$ openxc-control write --bus 1 --id 42 --data 0x1234
```

This brings us back from translated JSON to raw CAN hacking, as described earlier in this chapter. However, if you want to write an app or embedded graphical interface to only read and react to your vehicle and you own a new Ford, then this may be the quickest route to those goals.
```
