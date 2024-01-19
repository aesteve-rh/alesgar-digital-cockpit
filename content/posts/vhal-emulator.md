---
title: "Vhal Emulator"
date: 2024-01-19T14:00:47+01:00
draft: false
---

In this post I would like to introduce you the [VHAL Emulator](https://github.com/aesteve-rh/vhal_emulator) Rust library and
how to use it. But before that, I need to talk about Google's
[VHAL Host Emulator](https://android.googlesource.com/platform/packages/services/Car/+/master/tools/emulator/). The Rust crate is strongly based on the python modules
from Google, and leverages the same strategy, the same VHAL module, as their
python counterpart. So if you knew the python emulator, you have half the work
done for the Rust library.

But for those that are not familiar with this, I'll give you a few hints before
going too deep.

## Vehicle HAL

Firstly, [VHAL](https://source.android.com/docs/automotive/vhal) stands for
Vehicle Hardware Abstraction Layer, and it establishes a standardized approach
for accessing vehicle [properties](https://developer.android.com/reference/android/car/VehiclePropertyIds) in Android platforms. This way, devices and
third-party application can modify or query properties and interact seemingly.

These properties are defined with [HIDL](https://source.android.com/docs/core/architecture/hidl) or [AIDL](https://developer.android.com/develop/background-work/services/aidl)
files, although for new VHAL implementations, AIDL usage is strongly recommended.
These definitions allow both the server and all clients to have a common
interface with all the available properties (e.g., `vehicle speed` or `gear`), and
their underlying types (e.g., `float` or `int32`).

Note: See `hardware/interfaces/automotive/vehicle/2.0/types.hal`.

## Emulating VHAL communication

But what if you are developing a new Android App or want to add a new property
to the AIDL definition and would like to test it beforehand? Well, that is where
the VHAL Host Emulator comes in handy.
- On the device side, VehicleService creates
    a VehicleEmulator to setup SocketComm to serve the requests.
- On the host side, it uses [Android Debug Bridge](https://developer.android.com/develop/background-work/services/aidl) (adb) port forwarding to allow a socket to connect to the VHAL.

To make the properties consistent in both sides of the adb socket, you need to make
sure that you have the same HIDL/AIDL definition file on your host, and run
a code generator module included in the emulator folder. The code generator will
parse the file and generate code accordingly.

After that, it is simply a matter of sending the data through the socket with
the right format. In this case, the format is defined by Google's [Protocol
Buffer](https://github.com/protocolbuffers/protobuf) (i.e., protobuf)
file. This file can change over different Android versions, so make sure
you have an updated version or the messages will not be correctly
formatted to be read by the Vehicle HAL.

Note: see `hardware/interfaces/automotive/vehicle/2.0/default/impl/vhal_v2_0/proto/VehicleHalProto.proto`.

With both sides aligned and the class at [vhal_emulator.py](https://android.googlesource.com/platform/packages/services/Car/+/refs/heads/main/tools/emulator/vhal_emulator.py), you
can easily write a python script that interacts with the VHAL, and see how those
changes affect your new and awesome new App, or verify that your new AIDL definition
is correct.

## VHAL Emulator Rust library

So with all this, you have the necessary context to see the use for
https://github.com/aesteve-rh/vhal_emulator.

### Internal definitions

The library has its own copy of the [protobuf definition file](https://github.com/aesteve-rh/vhal_emulator/blob/main/src/protos/VehicleHalProto.proto), which is used
to generate Rust bindings at build time by using the [protobuf_codegen](https://docs.rs/protobuf-codegen/latest/protobuf_codegen/) crate.

Furthermore, it also has a copy of the
[vehicle hal definition file](https://github.com/aesteve-rh/vhal_emulator/blob/main/src/codegen/data/types.hal) in HIDL format, which is used to auto generate the `enums` for all
available Vehicle Properties. In this case, codegen need to be triggered
manually on definition file updates. To do this, there is a specific python
script, that uses a parser borrowed from the VHAL Host Emulator and
[Mako](https://www.makotemplates.org/) templates to generate the code based
on the definitions.

Luckily, there is a recipe to allow updating the generated code with ease:

```
$ make venv
$ source $HOME/.venv/vhal_emulator/bin/activate
$ make render
$ deactivate
```

### Using the library

The library functionality is mostly contained in the `Vhal` struct, with all
the helper methods to interact with the VHAL interface.

As mentioned above, the rust library uses the same adb port forwarding strategy
as the VHAL Host Emulator in python. You can open the port yourself using
adb and then specify the port to the `Vhal` struct when instantiating it.

However, for those that want a more programmatic solution, there is a helper
function available in the library:

```rust
use vhal_emulator as ve;

let local_port: u16 = ve::adb_port_forwarding().expect("Could not run adb for port forwarding");
let v = ve::Vhal::new(local_port).unwrap();
```

Currently, there are only few helper methods for specific properties:

- `[get|set]_gear_selection` to interact with [GEAR_SELECTION](https://developer.android.com/reference/android/car/VehiclePropertyIds#GEAR_SELECTION)
- `[get|set]_vehicle_speed` for [PERF_VEHICLE_SPEED](https://developer.android.com/reference/android/car/VehiclePropertyIds#PERF_VEHICLE_SPEED)
- `[get|set]_vehicle_display_speed` for [PERF_VEHICLE_SPEED_DISPLAY](https://developer.android.com/reference/android/car/VehiclePropertyIds#PERF_VEHICLE_SPEED_DISPLAY)

Note that these getters return data is based in their underlying type.

Of course, contributions are more than welcome to keep improving the library.
But worry not, there are a few generic methods that allow to send and receive
messages to interact with any property you need (e.g., if you have added
a new property and want to test it).

```rust
use vhal_emulator::vhal_consts_2_0 as c;

// Get the property config
v.get_config(c::VehicleProperty::HVAC_TEMPERATURE_SET).unwrap();

// Get the response message to get_config
let reply = v.recv_cmd().unwrap();

// Set left temperature to 70 degrees
v.set_property(
    c::VehicleProperty::HVAC_TEMPERATURE_SET,
    ve::VehiclePropertyValue::Int32(70),
    c::VehicleArea::SEAT | c:VehicleAreaSeat::ROW_1_LEFT,
    None,
).unwrap();

// Get the response message to set_property
let reply = v.recv_cmd().unwrap();
```

For example, this is something I did using the VHAL Emulator library in Rust.
The VHAL communication-related code is only a handful of lines.

{{< youtube 2fleJrTKBNU >}}

## Future Work

There is a lot of work to do. The library is still immature, with lots of room
for improvement. As mentioned above, there are very few methods for specific
property manipulation, which would make the library more attractive for
users, as they would get abstracted of all the data they do not need.

Currently, only HIDL format is supported. Thus, a new AIDL parser will be
required sooner or later, as it is strongly advised for newer properties.

The project also needs a lot more testing and infrastructure, to ensure that
the library is solid and safe to use.
Proper pipelines and containerization are also in the radar.
Even the [issues](https://github.com/aesteve-rh/vhal_emulator/issues) for all
these things are missing!

But I hope to cover in the next months some of these gaps (hopefully with some
help from contributors!), and make the library mature enough to be published
at https://crates.io/.

## Conclusion

Hopefully this post has clarified the why and how this library may be
interesting for you. But as you can see, there is a lot of work to do yet.
So if you like the library and want to contribute, please do!