# libsweep

Low-level Scanse Sweep LiDAR library. Comes as C99 library `sweep.h` with optional C++11 header `sweep.hpp` on top of it.

### Quick Start

```bash
mkdir -p build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build .
sudo cmake --build . --target install
sudo ldconfig
```

If you don't have a Sweep device yet you can build a dummy `libsweep.so` always returning static point cloud data:

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release -DDUMMY=On
```

This dummy library is API and ABI compatible. Once your device arrives switch out the `libsweep.so` shared library and you're good to go.

For Windows users open a command prompt with administrative access:

```bash
mkdir build
cd build
cmake .. -G "Visual Studio 14 2015 Win64"
cmake --build . --config Release
cmake --build . --target install --config Release
```

The above command assumes Visual Studio 2015. If you have a different version installed, change the value. ie:

    Visual Studio 11 2012 Win64 = Generates Visual Studio 11 (VS 2012) project files for x64 architecture
    Visual Studio 12 2013 Win64 = Generates Visual Studio 12 (VS 2013) project files for x64 architecture
    Visual Studio 14 2015 Win64 = Generates Visual Studio 14 (VS 2015) project files for x64 architecture
    Visual Studio 15 2017 Win64 = Generates Visual Studio 15 (VS 2017) project files for x64 architecture

Additionally, the above commands assume you want to build a x64 (64bit) verison of the library. To build a x86 (32 bit) version, simply drop the `Win64`. i.e.:

    Visual Studio 14 2015 = Generates Visual Studio 14 (VS 2015) project files for x86 architecture

To build the dummy library add the dummy flag to the command:

```bash
cmake ..  -DDUMMY=On -G "Visual Studio 14 2015 Win64"
```

Then be sure to add the installation directories for the library and the header files to the environment `PATH` variable. For the above installation that would be something like `C:\Program Files\sweep\lib` for the library and `C:\Program Files\sweep\inlcude\sweep` for the headers. You may have to restart the computer before the changes take effect.

### Usage

- Include `<sweep/sweep.h>` for the C interface or `<sweep/sweep.hpp>` for the C++ interface.
- Link `libsweep.so` with `-lsweep`.

For example:

    gcc -Wall -Wextra -pedantic -std=c99 examples/example.c -lsweep
    g++ -Wall -Wextra -pedantic -std=c++11 examples/example.cc -lsweep

In addition, we provide CMake integration. In your `CMakeLists.txt`:

    FindPackage(Sweep REQUIRED)
    target_link_libraries(.. ${LIBSWEEP_LIBRARY})
    target_include_directories(.. ${LIBSWEEP_INCLUDE_DIR})

See [example.c](examples/example.c) and [example.cc](examples/example.cc) for a C and C++ example, respectively.


### libsweep

Before jumping into details, here are the library's main design ideas:

- Only opaque pointers (think typed `void *`) and plain C types in the API. This is how we accomplish ABI compatibility.
- No global state: `_s` context objects (`_s` since POSIX reserves `_t`) hold state and have to be passed explicitly.
- Make ownership and lifetimes explicit: all objects have to be `_construct()`ed for creation and successfully created objects have to be `_destruct()`ed for cleanup.
- Sane defaults and user-friendly API: when possible we provide `_simple()` functions e.g. doing serial port auto-detection.
- Fixed-size and signed integers for portability and runtime checking (think sanitizers).

Table of Contents:
- [Version and ABI Management](#version-and-abi-management)
- [Error Handling](#error-handling)
- [Device Interaction](#device-interaction)
- [Full 360 Degree Scan](#full-360-degree-scan)


#### Version And ABI Management

```c++
SWEEP_VERSION_MAJOR
```

Interface major version number.

```c++
SWEEP_VERSION_MINOR
```

Interface minor version number.

```c++
SWEEP_VERSION
```

Combined interface major and minor version number.

```c++
int32_t sweep_get_version(void)
```

Returns the library's version (see `SWEEP_VERSION`).
Used for ABI compatibility checks.

```c++
bool sweep_is_abi_compatible(void)
```

Returns true if the library is ABI compatible.
This check is done by comparing the interface header with the installed library's version.


#### Error Handling

```c++
SWEEP_ASSERT
```

Optionally define this for custom assertion handling.
Uses `assert` from `<assert.h>` by default.

```c++
sweep_error_s
```

Opaque type representing an error.
You can get a human readable representation using the `sweep_error_message` function.
You have to destruct an error object with `sweep_error_destruct`.

```c++
const char* sweep_error_message(sweep_error_s error)
```

Human readable representation for an error.

```c++
void sweep_error_destruct(sweep_error_s error)
```

Destructs an `sweep_error_s` object.


#### Device Interaction

```c++
sweep_device_s
```

Opaque type representing a Sweep device.
All direct device interaction happens on this type.

```c++
sweep_device_s sweep_device_construct_simple(const char* port, sweep_error_s* error)
```

Constructs a `sweep_device_s` based on a serial device port (e.g. `/dev/ttyUSB0` on Linux or `COM5` on Windows).
In case of error a `sweep_error_s` will be written into `error`.

```c++
sweep_device_s sweep_device_construct(const char* port, int32_t bitrate, sweep_error_s* error)
```

Constructs a `sweep_device_s` with explicit hardware configuration.
In case of error a `sweep_error_s` will be written into `error`.

```c++
void sweep_device_destruct(sweep_device_s device)
```

Destructs an `sweep_device_s` object.

```c++
void sweep_device_start_scanning(sweep_device_s device, sweep_error_s* error)
```

Signals the `sweep_device_s` to start scanning.
In case of error a `sweep_error_s` will be written into `error`.

```c++
void sweep_device_stop_scanning(sweep_device_s device, sweep_error_s* error)
```

Signals the `sweep_device_s` to stop scanning.
In case of error a `sweep_error_s` will be written into `error`.

```c++
int32_t sweep_device_get_motor_speed(sweep_device_s device, sweep_error_s* error)
```

Returns the `sweep_device_s`'s motor speed in Hz.
In case of error a `sweep_error_s` will be written into `error`.

```c++
void sweep_device_set_motor_speed(sweep_device_s device, int32_t hz, sweep_error_s* error)
```

Sets the `sweep_device_s`'s motor speed in Hz.
The device supports speeds of 1 Hz to 10 Hz.
In case of error a `sweep_error_s` will be written into `error`.

```c++
int32_t sweep_device_get_sample_rate(sweep_device_s device, sweep_error_s* error)
```

Returns the `sweep_device_s`'s sample rate in Hz.
In case of error a `sweep_error_s` will be written into `error`.

```c++
void sweep_device_set_sample_rate(sweep_device_s device, int32_t hz, sweep_error_s* error)
```

Sets the `sweep_device_s`'s sample rate in Hz.
The device supports sample rates of 500 Hz, 750 Hz and 1000 Hz.
The device guarantees for those sample rates but they can be slightly higher by a maximum of roughly 50-100 Hz.
In case of error a `sweep_error_s` will be written into `error`.

```c++
void sweep_device_reset(sweep_device_s device, sweep_error_s* error)
```

Resets the `sweep_device_s` hardware.
In case of error a `sweep_error_s` will be written into `error`.


#### Full 360 Degree Scan

```c++
sweep_scan_s
```

Opaque type representing a single full 360 degree scan from a `sweep_device_s`.

```c++
sweep_scan_s sweep_device_get_scan(sweep_device_s device, sweep_error_s* error)
```

Blocks waiting for the `sweep_device_s` to accumulate a full 360 degree scan into `sweep_scan_s`.
In case of error a `sweep_error_s` will be written into `error`.

```c++
void sweep_scan_destruct(sweep_scan_s scan)
```

Destructs a `sweep_scan_s` object.

```c++
int32_t sweep_scan_get_number_of_samples(sweep_scan_s scan)
```

Returns the number of samples in a full 360 degree `sweep_scan_s`.

```c++
int32_t sweep_scan_get_angle(sweep_scan_s scan, int32_t sample)
```

Returns the angle in milli-degree for the `sample`th sample in the `sweep_scan_s`.

```c++
int32_t sweep_scan_get_distance(sweep_scan_s scan, int32_t sample)
```

Returns the distance in centi-meter for the `sample`th sample in the `sweep_scan_s`.

```c++
int32_t sweep_scan_get_signal_strength(sweep_scan_s scan, int32_t sample)
```

Returns the signal strength (0 low -- 255 high) for the `sample`th sample in the `sweep_scan_s`.


### License

Copyright © 2016 Daniel J. Hofmann

Copyright © 2016 Scanse LLC

Distributed under the MIT License (MIT).
