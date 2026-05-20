# pico2w-microros-wifi-guide

A minimal working setup for running a [micro-ROS](https://micro.ros.org/) node on
the Raspberry Pi Pico 2W (or original Pico W) over Wi-Fi via UDP transport. The
example publishes an incrementing counter to the ROS 2 topic `/pico_counter`.

This repo is a small, self-contained example of a Wi-Fi-connected micro-ROS node
on the Pico 2W. Includes a working custom UDP transport and the radio
configuration needed for consistent latency.

For the performance data including expected message round-trip-time across serial vs. Wi-Fi
and ad-hoc vs. micro-ROS configurations, see the companion
repo: [`pico2w-microros-benchmark`](https://anonymous.4open.science/r/pico2w-microros-benchmark-7AD6/).

---

## Hardware

- Raspberry Pi Pico 2W (or original Pico W)
- Host machine running ROS 2 Jazzy on the same Wi-Fi network

---

## Files

| File | Purpose |
| --- | --- |
| `pico_micro_ros_example.c` | Main firmware: connects to Wi-Fi, sets up UDP transport, publishes a counter |
| `picow_udp_transports.c` / `.h` | Custom micro-ROS UDP transport for the CYW43439 radio |
| `lwipopts.h` | lwIP TCP/IP stack configuration |
| `CMakeLists.txt` | Build configuration |

---

## Prerequisites

Install build dependencies:

```bash
sudo apt install build-essential cmake gcc-arm-none-eabi libnewlib-arm-none-eabi doxygen git python3
```

Install the Pico SDK:

```bash
cd ~
git clone --recurse-submodules https://github.com/raspberrypi/pico-sdk.git
echo "export PICO_SDK_PATH=$HOME/pico-sdk" >> ~/.bashrc
source ~/.bashrc
```

Install the GCC ARM toolchain (9.3.1) — download from
[Arm's developer site](https://developer.arm.com/downloads/-/gnu-rm), extract
to your home directory, then:

```bash
echo "export PICO_TOOLCHAIN_PATH=$HOME/gcc-arm-none-eabi-9-2020-q2-update/" >> ~/.bashrc
source ~/.bashrc
```

---

## Configuration

### Board selection

This repo defaults to the Pico 2W. For the original Pico W, edit line 2 of
`CMakeLists.txt`:

```cmake
set(PICO_BOARD pico_w)    # original Pico W
# set(PICO_BOARD pico2_w) # Pico 2W (default)
```

### Wi-Fi credentials

Edit the SSID and password at the top of `pico_micro_ros_example.c`:

```c
char ssid[] = "SSID";      // your Wi-Fi network name
char pass[] = "PASSWORD";  // your Wi-Fi password
```

### Agent IP address

Set your host machine's IP address in `picow_udp_transports.h`:

```c
#define ROS_AGENT_IP_ADDR   "192.168.50.209"
```

You can find your host's IP with `hostname -I` or `ip addr show`.

---

## Build the libmicroros static library

`libmicroros` is a precompiled static library required by this project. It is
not included in the repo and must be generated once using Docker.

```bash
mkdir -p ~/micro_ros_ws/src
cd ~/micro_ros_ws/src
git clone https://github.com/micro-ROS/micro_ros_raspberrypi_pico_sdk.git
cd micro_ros_raspberrypi_pico_sdk
docker run --rm \
    -v $(pwd):/project \
    --env PICO_SDK_PATH=/project/pico-sdk \
    microros/micro_ros_static_library_builder:jazzy
```

Copy the output into this project:

```bash
cp -r ~/micro_ros_ws/src/micro_ros_raspberrypi_pico_sdk/libmicroros \
      <path-to-this-repo>/libmicroros
```

---

## Build

```bash
cd <path-to-this-repo>
mkdir build && cd build
cmake ..
make
```

The build produces `pico_micro_ros_example.uf2` in the `build/` directory.

---

## Flash

Hold **BOOTSEL** on the Pico while plugging it in via USB. The board mounts as
a mass-storage device, then:

**Pico W:**
```bash
cp build/pico_micro_ros_example.uf2 /media/$USER/RPI-RP2
```

**Pico 2W:**
```bash
cp build/pico_micro_ros_example.uf2 /media/$USER/RP2350
```

The Pico will reboot automatically and begin running the firmware.

---

## Run the micro-ROS agent

On the host machine:

```bash
sudo docker run -it --rm --net=host microros/micro-ros-agent:jazzy udp4 -p 8888
```

The Pico will retry connecting to the agent for up to ~60 seconds after
startup.

Verify the topic is live:

```bash
ros2 topic list
ros2 topic echo /pico_counter
```

You should see an incrementing `data:` value once per second.

---

## Notes

### Power-save mode

By default this firmware disables the CYW43439 radio's power-save mode
(`cyw43_wifi_pm(&cyw43_state, CYW43_NO_POWERSAVE_MODE)` near the top of
`main()` in `pico_micro_ros_example.c`).

The radio's default mode puts the chip to sleep after a 200 ms idle timer.
For periodic communication at intervals slower than ~5 Hz, every message
incurs a radio wake-up penalty of roughly 80 ms. Disabling power save keeps
the radio active and produces consistent latency across message rates.

If you need to minimise power draw and don't require consistent latency,
comment out that line.

### Wi-Fi transport

The custom UDP transport in `picow_udp_transports.c` uses a ring buffer to
decouple the lwIP/CYW43 receive callback from the main thread. Comments in
that file explain the concurrency model in detail. You shouldn't need to
modify it for typical applications.

---

## Dependencies

- [micro-ROS Raspberry Pi Pico SDK](https://github.com/micro-ROS/micro_ros_raspberrypi_pico_sdk)
- [Raspberry Pi Pico SDK](https://github.com/raspberrypi/pico-sdk)
- ROS 2 Jazzy

---

## License

MIT — see [LICENSE](LICENSE).
