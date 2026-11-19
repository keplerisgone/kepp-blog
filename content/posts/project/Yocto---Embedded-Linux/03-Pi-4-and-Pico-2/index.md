---
title: "03: UART Communication Between Yocto Linux and Raspberry Pi Pico 2"
date: "2026-11-19T00:00:00+09:00"
lastmod: "2026-11-19T00:00:00+09:00"
draft: false
description: "Connect Raspberry Pi 4 running Yocto Linux to Pico 2 over UART and verify a newline-delimited PING/PONG protocol."
categories: ["project", "Yocto---Embedded-Linux"]
tags: ["Yocto", "Embedded Linux", "Raspberry Pi", "Pico 2", "UART", "Device Tree"]
weight: 3
ShowToc: true
---

## 1. Introduction

In previous experiments, I built a custom Embedded Linux image using the Yocto Project, deployed it to a Raspberry Pi 4, and established a serial console through a USB-UART adapter.

I also implemented a UART loopback test in C using `termios`, `read()`, `write()`, and `poll()`.

However, loopback testing only verifies communication through a single UART interface.

The objective of this experiment was to establish communication between two independent devices:

- **Raspberry Pi 4:** Running a custom Yocto Linux image.
- **Raspberry Pi Pico 2:** Running bare-metal C firmware.

The final goal was to implement a simple request-response protocol:

```text
Raspberry Pi 4  →  Pico 2: PING
Raspberry Pi 4  ←  Pico 2: PONG
```

The experiment successfully received a `PONG` response from the Pico 2.

## 2. System Architecture

The system contains three main components:

1. A Linux development PC running Omarchy.
2. A Raspberry Pi 4 running Yocto Linux.
3. A Raspberry Pi Pico 2 running custom firmware.

The development PC communicates with the Raspberry Pi through a PL2303 USB-UART adapter.

A separate UART interface connects the Raspberry Pi to the Pico 2.

```text
Development PC (Omarchy)
          |
          | USB
          v
   PL2303 USB-UART
          |
          | UART console
          v
+-------------------------+
| Raspberry Pi 4          |
|                         |
| Yocto Linux             |
|                         |
| /dev/ttyS0              |
|   - Debugging console   |
|                         |
| /dev/ttyAMA3            |
|   - Application UART    |
+-------------------------+
          |
          | UART3: 115200 8N1
          |
          v
+-------------------------+
| Raspberry Pi Pico 2     |
|                         |
| RP2350 MCU              |
|                         |
| Bare-metal C firmware   |
| UART0                   |
+-------------------------+
```

### 2.1 Different Software Environments

Although both devices use UART, their programming environments are fundamentally different.

| Component | Raspberry Pi 4 | Pico 2 |
|---|---|---|
| Processor | ARM Cortex-A72 | RP2350 MCU |
| Execution environment | Linux | Bare-metal |
| UART interface | `/dev/ttyAMA3` | `uart0` |
| Software interface | Linux TTY | Pico SDK |
| I/O functions | `read()`, `write()` | `uart_getc()`, `uart_puts()` |
| Hardware access | Through kernel drivers | Through MCU peripheral APIs |

The Raspberry Pi uses the Linux kernel to abstract UART hardware.

The Pico 2, on the other hand, executes firmware that communicates with its UART peripheral through the Pico SDK.

Both environments ultimately transmit and receive bytes using compatible UART signaling.

## 3. Raspberry Pi UART Configuration

### 3.1 Why a Separate UART Was Necessary

The Raspberry Pi already used `/dev/ttyS0` as its Linux serial console.

The kernel boot parameters included:

```text
console=ttyS0,115200
```

The `serial-getty@ttyS0.service` service provided an interactive login prompt.

Using the same UART for application communication could cause interference between console traffic and application data.

Therefore, UART3 was selected as a separate interface.

### 3.2 Enabling UART3

The Raspberry Pi boot configuration was modified by adding the following Device Tree overlay:

```ini
dtoverlay=uart3
```

The existing UART console configuration was preserved.

After rebooting, the Linux kernel reported:

```text
fe201600.serial: ttyAMA3 at MMIO 0xfe201600
    (irq = 38, base_baud = 0) is a PL011 rev3
```

The device node was confirmed using:

```bash
ls -l /dev/ttyAMA3
```

This confirmed that the ARM PL011 driver had registered the additional UART controller.

### 3.3 Device Tree and Driver Binding

The Device Tree describes hardware resources that the kernel cannot necessarily discover automatically.

For UART peripherals, this includes register addresses, interrupts, clocks, and pin configurations.

In this experiment:

```text
Device Tree Overlay
        |
        v
Enable UART3
        |
        v
ARM PL011 Driver
        |
        v
/dev/ttyAMA3
        |
        v
Linux Userspace
```

The UART3 controller was mapped at the physical MMIO address `0xfe201600`.

This provided a practical example of how Device Tree configuration, memory-mapped I/O, kernel drivers, and character device nodes relate to each other.

## 4. Hardware Connections

The Raspberry Pi 4 and Pico 2 were connected using UART TX, RX, and GND.

![Raspberry Pi 4 and Pico 2 connected by jumper wires](image%20%281%29.jpg)

*Overview of the Raspberry Pi 4 and Pico 2 UART wiring.*

| Raspberry Pi 4 | Pico 2 | Purpose |
|---|---|---|
| GPIO4, physical pin 7 | GP1, physical pin 2 | Pi TX → Pico RX |
| GPIO5, physical pin 29 | GP0, physical pin 1 | Pi RX ← Pico TX |
| GND, physical pin 9 | GND, physical pin 3 | Common ground |

The connection follows the standard UART rule:

```text
Raspberry Pi 4              Pico 2

UART3 TX  ----------------> UART0 RX
UART3 RX  <---------------- UART0 TX
GND       ----------------- GND
```

Both devices use 3.3V UART signaling.

Each board was powered independently through USB, and the power rails were not connected together.

![Pico 2 UART header connections](image%20%282%29.jpg)

*Close-up of the Pico 2 and the jumper connections used in the experiment.*

**Important:** The GPIO connections should be made with both boards powered off. GPIO pins must not be exposed to incompatible voltage levels.

## 5. Designing the UART Protocol

UART provides serial byte transmission, but it does not define the structure of application messages.

Therefore, a simple text-based protocol was implemented.

| Command | Response |
|---|---|
| `PING\n` | `PONG\n` |

The protocol uses a newline character to mark the end of a message.

For example:

```text
'P' 'I' 'N' 'G' '\n'
```

This is necessary because UART is a byte-stream interface. Receiving four characters does not automatically tell the receiver whether the message is complete.

The Pico firmware accumulates incoming characters until it receives a newline.

It then compares the resulting string with `PING`.

If the command matches, it transmits `PONG\n`.

## 6. Implementing Pico 2 Firmware

The Pico firmware was developed in C using the Raspberry Pi Pico SDK.

### 6.1 Firmware Implementation

```c
#include <string.h>

#include "pico/stdlib.h"
#include "hardware/uart.h"

#define UART_ID   uart0
#define BAUD_RATE 115200

int main(void)
{
    uart_init(UART_ID, BAUD_RATE);

    gpio_set_function(0, GPIO_FUNC_UART);
    gpio_set_function(1, GPIO_FUNC_UART);

    uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE);
    uart_set_hw_flow(UART_ID, false, false);

    char buffer[16];
    size_t pos = 0;

    while (true) {
        char c = uart_getc(UART_ID);

        if (c == '\r')
            continue;

        if (c == '\n') {
            buffer[pos] = '\0';

            if (strcmp(buffer, "PING") == 0)
                uart_puts(UART_ID, "PONG\n");

            pos = 0;
        } else if (pos < sizeof(buffer) - 1) {
            buffer[pos++] = c;
        } else {
            pos = 0;
        }
    }
}
```

### 6.2 Understanding the Firmware

The initialization function:

```c
uart_init(UART_ID, BAUD_RATE);
```

configures the UART peripheral with the requested baud rate.

The following functions assign the GPIO pins to UART operation:

```c
gpio_set_function(0, GPIO_FUNC_UART);
gpio_set_function(1, GPIO_FUNC_UART);
```

The serial format is configured using:

```c
uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE);
```

This selects eight data bits, one stop bit, and no parity.

Hardware flow control is disabled:

```c
uart_set_hw_flow(UART_ID, false, false);
```

The main loop receives characters using:

```c
char c = uart_getc(UART_ID);
```

This is a blocking operation. The function waits until a character arrives.

Once a newline is received, the firmware terminates the string and compares it with the expected command.

```c
if (strcmp(buffer, "PING") == 0)
    uart_puts(UART_ID, "PONG\n");
```

This simple implementation is sufficient for testing basic communication.

However, a more advanced protocol would need explicit error handling, timeout behavior, and message validation.

## 7. Building the Pico Firmware

The firmware was compiled with CMake and the Pico SDK.

### 7.1 CMake Configuration

```cmake
cmake_minimum_required(VERSION 3.13)

include(pico-sdk/pico_sdk_init.cmake)

project(uart_pong C CXX ASM)

pico_sdk_init()

add_executable(uart_pong main.c)

target_link_libraries(uart_pong
    pico_stdlib
    hardware_uart
)

pico_add_extra_outputs(uart_pong)
```

The build was configured for the Pico 2:

```bash
cmake -S . -B build -G Ninja \
  -DPICO_BOARD=pico2
```

Then:

```bash
cmake --build build
```

The expected firmware artifact was:

```text
build/uart_pong.uf2
```

Unlike a Linux executable, a UF2 image contains firmware intended to run on the microcontroller.

## 8. Flashing Firmware Using BOOTSEL

The Pico 2 supports firmware installation through its built-in USB bootloader.

The procedure was:

1. Disconnect the Pico 2 from USB.
2. Hold the BOOTSEL button.
3. Reconnect the Micro-USB cable while holding BOOTSEL.
4. Release the button.
5. Identify the Pico's USB mass-storage device on the development PC.
6. Copy `uart_pong.uf2` to the device.

On Linux, the device can be located using:

```bash
lsblk -o NAME,LABEL,SIZE,MOUNTPOINTS
```

The Pico 2 BOOTSEL volume normally appears with the label `RP2350`.

Once mounted, the firmware can be copied using a normal file-copy command:

```bash
cp build/uart_pong.uf2 /path/to/RP2350/
```

After the transfer, the Pico 2 normally disconnects the USB mass-storage interface and starts executing the uploaded firmware.

The firmware does not produce visible output over USB because its communication interface is UART0 on GP0 and GP1.

## 9. Testing UART Communication from Yocto Linux

With the firmware running and the GPIO wiring connected, the Raspberry Pi was used to send commands.

### 9.1 Configuring the UART

The Raspberry Pi exposes UART3 as:

```text
/dev/ttyAMA3
```

The port was configured with:

```bash
stty -F /dev/ttyAMA3 115200 raw -echo -crtscts
```

The options mean:

| Option | Meaning |
|---|---|
| `115200` | Set baud rate |
| `raw` | Disable terminal-oriented character processing |
| `-echo` | Disable input echo |
| `-crtscts` | Disable hardware flow control |

### 9.2 Receiving Data

A background receiver was started:

```bash
cat /dev/ttyAMA3 &
```

This process reads incoming data from the UART device and prints it to the console.

### 9.3 Transmitting the Command

The command was transmitted using:

```bash
printf 'PING\n' > /dev/ttyAMA3
```

Unlike manually typing into a serial terminal, `printf` explicitly sends the newline character expected by the firmware.

### 9.4 Actual Result

The Raspberry Pi successfully received:

```text
PONG
```

**Result: PASS**

This verifies successful request-response communication between the Raspberry Pi 4 and Pico 2 using the tested UART configuration.

The test exercised both transmission directions:

```text
Raspberry Pi 4                    Pico 2
     |                               |
     | -------- PING\n ------------> |
     |                               |
     | <------- PONG\n ------------- |
     |                               |
```

After the experiment, the temporary background receiver can be stopped using:

```bash
kill %1
```

## 10. Troubleshooting: No Response from the Pico 2

During the initial test, no response was received from the Pico 2.

Several possible causes were considered:

- Incorrect TX/RX wiring.
- Missing common ground.
- Firmware not executing on the Pico.
- Incorrect UART baud rate.
- Missing message delimiter.
- Incorrect Linux UART device.

An important detail was the use of newline characters.

The Pico firmware only processes a command when it receives `\n`.

Using a terminal program can introduce ambiguity because the Enter key might transmit `\r`, `\n`, or both, depending on the terminal configuration.

Although the Pico firmware ignores carriage returns, it still requires a newline to complete the message.

Therefore, the test was changed to use:

```bash
printf 'PING\n' > /dev/ttyAMA3
```

After explicitly transmitting the expected byte sequence, the Pico responded with `PONG`.

This demonstrated an important distinction between **physical UART connectivity and application-level protocol correctness**.

A UART connection can be electrically functional while the application still fails to respond because the transmitted message does not match the protocol.

The successful `printf` test supports a line-ending mismatch as a plausible explanation for the earlier failure, although the initial failure's exact cause was not independently isolated.

## 11. Lessons Learned

### 11.1 UART Is a Byte Stream

UART does not inherently understand commands, packets, or messages.

Applications must define framing rules.

For this experiment, the newline character served as the message delimiter.

### 11.2 Device Tree Connects Hardware to Linux Drivers

The UART3 peripheral became available only after enabling the relevant Device Tree overlay.

The resulting device was registered by the Linux PL011 driver and exposed through `/dev/ttyAMA3`.

This connected the concepts of Device Tree, MMIO, interrupts, device drivers, and userspace interfaces.

### 11.3 Linux and Bare-metal Firmware Use Different Abstractions

The Raspberry Pi communicates through the Linux TTY subsystem.

The Pico 2 directly configures its MCU UART peripheral using the Pico SDK.

Despite these differences, the two devices can communicate because their UART configurations and application protocol agree.

### 11.4 Protocol Details Matter

UART connectivity alone is not sufficient.

Both endpoints must agree on:

- Baud rate and serial format.
- Message boundaries.
- Command syntax.
- Response format.
- Error handling.

A missing newline was enough to become a central troubleshooting consideration.

## 12. Conclusion

In this experiment, I successfully established UART communication between a Raspberry Pi 4 running Yocto Linux and a Raspberry Pi Pico 2 running bare-metal C firmware.

The Raspberry Pi used a dedicated UART3 interface, leaving its existing serial console available for debugging.

The Pico 2 firmware received `PING\n` and transmitted `PONG\n`.

The successful response confirmed that application-level communication worked across both hardware and software environments.

The resulting architecture is:

```text
Raspberry Pi 4
    |
    | Linux userspace
    | write() / read()
    v
Linux TTY Subsystem
    |
    v
ARM PL011 Driver
    |
    v
UART3 Hardware
    |
    | TX / RX / GND
    v
Pico 2 UART0
    |
    v
Pico SDK Firmware
    |
    v
PING/PONG Protocol
```

This experiment established the foundation for more complex embedded communication.

The next development stage is to implement a dedicated C application on the Raspberry Pi, integrate it into a custom Yocto image using a BitBake recipe, and improve the protocol with explicit framing, timeout handling, and error detection.

## References

- [Raspberry Pi — UART Documentation](https://www.raspberrypi.com/documentation/computers/configuration.html#uart)
- [Raspberry Pi — Pico C/C++ SDK](https://www.raspberrypi.com/documentation/microcontrollers/c_sdk.html)
- [Raspberry Pi — Pico Series Documentation](https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html)
- [Linux Kernel — Device Tree Documentation](https://docs.kernel.org/devicetree/)
- [Linux Kernel — TTY Documentation](https://docs.kernel.org/driver-api/tty/)
- [Yocto Project Documentation](https://docs.yoctoproject.org/6.0/)
