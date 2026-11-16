---
title: "02: Linux UART Programming - termios, poll, and Loopback Testing"
date: "2026-11-16T00:00:00+09:00"
lastmod: "2026-11-16T00:00:00+09:00"
draft: false
description: "Implement a Linux UART loopback application in C using termios, read, write, and poll, and verify timeout handling."
categories: ["project", "Yocto---Embedded-Linux"]
tags: ["Yocto", "Embedded Linux", "UART", "TTY", "C", "termios", "poll"]
weight: 2
ShowToc: true
---

## 1. Objective

The objective of this experiment is to understand how a Linux userspace application communicates with UART hardware through the TTY subsystem.

In previous experiments, I built a custom Linux image using Yocto, booted it on a Raspberry Pi 4, and accessed its serial console through a PL2303 USB-to-UART adapter.

This time, instead of using a terminal application such as `picocom`, I implemented my own UART communication program in C.

The experiment focuses on four objectives:

1. Configure UART communication using `termios`.
2. Transmit and receive data using `write()` and `read()`.
3. Wait for incoming data using `poll()`.
4. Verify successful communication and timeout handling through loopback testing.

## 2. UART and the Linux TTY Subsystem

UART (Universal Asynchronous Receiver/Transmitter) is a hardware interface that enables asynchronous serial communication.

A typical UART connection uses three signals:

| Signal | Function |
|---|---|
| TX | Transmits data |
| RX | Receives data |
| GND | Common electrical reference |

Both devices must agree on communication parameters such as baud rate, data bits, parity, and stop bits.

For this experiment, I used the following configuration:

| Parameter | Value |
|---|---|
| Baud rate | 115200 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Hardware flow control | Disabled |

This configuration is commonly called **115200 8N1**.

### 2.1 Linux Device Abstraction

Linux exposes serial interfaces through character device files.

In this experiment, the PL2303 adapter appeared as:

```text
/dev/ttyUSB0
```

The communication path is:

```text
C Application
    |
    | open(), read(), write(), ioctl()
    v
Linux System Call Interface
    |
    v
TTY Subsystem
    |
    v
USB Serial Driver (pl2303)
    |
    v
PL2303 Hardware
    |
    v
UART TX/RX
```

The application does not need to manipulate UART hardware registers directly.

Instead, the kernel driver manages the hardware and exposes a standardized file interface.

This is an example of hardware abstraction in Linux.

## 3. Configuring UART with termios

Opening a serial device returns a file descriptor:

```c
int fd = open("/dev/ttyUSB0", O_RDWR | O_NOCTTY);
```

- `O_RDWR`: Open for reading and writing.
- `O_NOCTTY`: Prevent the device from becoming the process's controlling terminal.

However, opening the device does not automatically establish the required UART communication settings.

Linux provides the `termios` interface for this purpose.

### 3.1 Reading the Existing Configuration

```c
struct termios tty;

tcgetattr(fd, &tty);
```

`tcgetattr()` retrieves the current terminal settings.

### 3.2 Enabling Raw Mode

```c
cfmakeraw(&tty);
```

Terminals normally perform character processing, such as:

- Line buffering in canonical mode.
- Input echo.
- Special-character handling.
- Input and output character transformations.

These behaviors are useful for interactive terminals but generally undesirable for binary communication protocols.

Raw mode disables most of these transformations.

As a result, the application can process incoming UART bytes without terminal-specific interpretation.

### 3.3 Configuring 8N1

```c
tty.c_cflag |= CLOCAL | CREAD;
tty.c_cflag &= ~(CRTSCTS | PARENB | CSTOPB | CSIZE);
tty.c_cflag |= CS8;
```

The configuration flags mean:

| Flag | Meaning |
|---|---|
| `CLOCAL` | Ignore modem control lines |
| `CREAD` | Enable reception |
| `CRTSCTS` | Hardware RTS/CTS flow control |
| `PARENB` | Enable parity |
| `CSTOPB` | Use two stop bits when set |
| `CSIZE` | Character-size bit mask |
| `CS8` | Select eight data bits |

Clearing `PARENB` disables parity, and clearing `CSTOPB` selects one stop bit.

### 3.4 Configuring Baud Rate

```c
cfsetispeed(&tty, B115200);
cfsetospeed(&tty, B115200);
```

These functions configure the input and output speeds.

The new settings are applied using:

```c
tcsetattr(fd, TCSANOW, &tty);
```

`TCSANOW` requests that the settings take effect immediately.

### 3.5 Configuring Read Behavior

```c
tty.c_cc[VMIN] = 0;
tty.c_cc[VTIME] = 0;
```

In noncanonical mode, `VMIN` and `VTIME` influence how `read()` behaves.

Setting both to zero allows `read()` to return immediately with the available data, including zero bytes when none are available.

In this experiment, `poll()` is responsible for waiting until the serial device becomes readable.

## 4. Blocking I/O and poll()

A fundamental problem in serial communication is determining when data becomes available.

A blocking `read()` may wait for incoming bytes, depending on the descriptor flags and terminal settings.

Alternatively, an application could repeatedly check for data:

```c
while (1) {
    read(fd, buffer, sizeof(buffer));
}
```

Without appropriate waiting or rate limiting, this can lead to busy polling and unnecessary CPU consumption.

Linux provides `poll()` to wait for I/O readiness.

### 4.1 Using poll()

```c
struct pollfd pfd = {
    .fd = fd,
    .events = POLLIN
};

int ret = poll(&pfd, 1, 2000);
```

The parameters specify:

- `&pfd`: An array containing the file descriptors to monitor.
- `1`: The number of descriptors.
- `2000`: Timeout in milliseconds.

The return value has three cases:

| Return value | Meaning |
|---|---|
| Positive | An event occurred |
| Zero | Timeout expired |
| Negative | Error or interruption |

After a positive return, the application examines `pfd.revents` to determine which event occurred.

For example:

```c
if (pfd.revents & POLLIN) {
    read(fd, buffer, sizeof(buffer));
}
```

Unlike repeatedly calling `read()` in a busy loop, `poll()` allows the process to sleep until an event occurs or the timeout expires.

This concept is also used in Linux socket programming and other event-driven applications.

## 5. UART Loopback Experiment

To verify UART communication without requiring another device, I performed a loopback test using the PL2303 adapter.

### 5.1 Hardware Setup

![USB-to-UART adapter and jumper wires used for the loopback experiment](image%20%281%29.jpg)

*USB-to-UART adapter and jumper wires used in the experiment; TX and RX are connected for the positive loopback test.*

The adapter's TX and RX pins were connected directly.

```text
Development PC
      |
      | USB
      v
PL2303 Adapter
      |
      +---- TX ----+
      |            |
      +---- RX ----+
```

The USB connection powers the adapter.

VCC was not connected to an external device.

This experiment tests the transmit and receive paths by sending bytes out through TX and receiving them through RX.

### 5.2 Test Procedure

The application performs the following operations:

1. Open `/dev/ttyUSB0`.
2. Configure UART using `termios`.
3. Send the string `HELLO`.
4. Wait for incoming data using `poll()`.
5. Read the received bytes.
6. Compare the received data with the transmitted data.
7. Report success or failure.

The important transmission operation is:

```c
const char msg[] = "HELLO";

write(fd, msg, sizeof(msg) - 1);
```

The program then waits for a readable event:

```c
struct pollfd pfd = {
    .fd = fd,
    .events = POLLIN
};

poll(&pfd, 1, 2000);
```

It collects the expected number of bytes before comparing the result.

This is important because `read()` does not guarantee that an entire application-level message will arrive in one call.

## 6. Implementation

The following program is the UART loopback implementation used in the lesson.

```c
#include <errno.h>
#include <fcntl.h>
#include <poll.h>
#include <stdio.h>
#include <string.h>
#include <termios.h>
#include <unistd.h>

int main(void)
{
    int fd = open("/dev/ttyUSB0", O_RDWR | O_NOCTTY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    struct termios tty;
    if (tcgetattr(fd, &tty) < 0) {
        perror("tcgetattr");
        close(fd);
        return 1;
    }

    cfmakeraw(&tty);
    tty.c_cflag |= CLOCAL | CREAD;
    tty.c_cflag &= ~(CRTSCTS | PARENB | CSTOPB | CSIZE);
    tty.c_cflag |= CS8;

    tty.c_cc[VMIN] = 0;
    tty.c_cc[VTIME] = 0;

    if (cfsetispeed(&tty, B115200) < 0 ||
        cfsetospeed(&tty, B115200) < 0 ||
        tcsetattr(fd, TCSANOW, &tty) < 0) {
        perror("termios");
        close(fd);
        return 1;
    }

    tcflush(fd, TCIFLUSH);

    const char msg[] = "HELLO";
    const size_t len = sizeof(msg) - 1;

    if (write(fd, msg, len) != (ssize_t)len) {
        perror("write");
        close(fd);
        return 1;
    }

    struct pollfd pfd = {
        .fd = fd,
        .events = POLLIN
    };

    char buf[sizeof(msg)] = {0};
    size_t received = 0;

    while (received < len) {
        int ret = poll(&pfd, 1, 2000);

        if (ret < 0) {
            if (errno == EINTR)
                continue;
            perror("poll");
            break;
        }

        if (ret == 0) {
            fprintf(stderr, "UART timeout\n");
            break;
        }

        if (pfd.revents & (POLLERR | POLLHUP | POLLNVAL)) {
            fprintf(stderr, "UART device error\n");
            break;
        }

        if (!(pfd.revents & POLLIN))
            continue;

        ssize_t n = read(fd, buf + received, len - received);

        if (n < 0 && errno == EINTR)
            continue;

        if (n <= 0) {
            if (n < 0)
                perror("read");
            break;
        }

        received += (size_t)n;
    }

    int ok = received == len &&
             memcmp(buf, msg, len) == 0;

    printf("Loopback: %s\n", ok ? "PASS" : "FAIL");

    close(fd);
    return ok ? 0 : 1;
}
```

This implementation is intended for a short educational experiment. It handles partial reads and interrupted `poll()` calls, but a production implementation should also handle partial writes and enforce an overall transaction deadline.

## 7. Build and Test Results

The program was compiled using GCC:

```bash
gcc -Wall -Wextra -O2 \
    uart_loopback.c -o uart_loopback
```

### Test 1 — TX and RX Connected

With TX connected directly to RX:

```bash
./uart_loopback
```

Actual output:

```text
Loopback: PASS
```

The transmitted bytes were successfully received and matched the original message.

This confirms that the UART transmit and receive paths were operational in the tested configuration.

### Test 2 — TX and RX Disconnected

Next, the physical loopback connection was removed.

The program was executed again:

```bash
./uart_loopback
```

Actual output:

```text
UART timeout
Loopback: FAIL
```

Although the output contains `FAIL`, this is the expected result for the negative test.

The receiver did not receive the transmitted message, and the program detected the absence of incoming data through a timeout.

This demonstrates that the application can handle a basic communication failure without waiting indefinitely.

### Test Summary

| Test | Expected | Actual |
|---|---|---|
| UART loopback connected | PASS | PASS |
| UART loopback disconnected | Timeout | Timeout |
| Program compilation | No compiler warnings | No warnings observed |

Both test scenarios behaved as intended.

## 8. Important Observations

### 8.1 write() Does Not Guarantee Physical Transmission Completion

A successful `write()` generally indicates that the kernel accepted bytes for output.

The bytes may still be waiting in a software or hardware transmission buffer.

For a serial terminal device, `tcdrain()` can be used to wait for queued output to finish transmitting.

This matters when a communication protocol requires precise sequencing.

### 8.2 poll() Does Not Guarantee a Complete Message

`poll()` indicates I/O readiness, not message completeness.

Even after `POLLIN` is reported, `read()` may return fewer bytes than requested.

Therefore, serial applications must define how messages are framed and how partial messages are handled.

### 8.3 UART Does Not Define an Application Protocol

UART provides byte transmission, but it does not define:

- Message boundaries.
- Commands and responses.
- Checksums.
- Retransmission rules.
- Application-level acknowledgments.

These features must be implemented by a higher-level protocol when required.

For example, a future Raspberry Pi–Pico communication protocol could exchange messages such as:

```text
Pi   -> Pico: PING
Pico -> Pi:   PONG
```

This would require both devices to agree on command formatting and message boundaries.

## 9. Conclusion

In this experiment, I implemented a UART communication program in C using the Linux TTY subsystem.

I configured the serial interface with `termios`, transmitted and received bytes through standard system calls, and used `poll()` to wait for incoming data.

The loopback experiment produced the expected results in both positive and negative test scenarios.

The key lesson was understanding that UART communication in Linux involves multiple layers:

```text
Application
    |
System Calls
    |
TTY Subsystem
    |
Serial Driver
    |
UART Hardware
```

The operating system abstracts the hardware so that userspace programs can communicate using file descriptors rather than directly manipulating device registers.

**Next step:** Move from local UART loopback testing to communication between the Raspberry Pi 4 and an external MCU, then integrate the C application into a custom Yocto image using a BitBake recipe.

## References

- [Linux termios(3) Manual](https://man7.org/linux/man-pages/man3/termios.3.html)
- [Linux poll(2) Manual](https://man7.org/linux/man-pages/man2/poll.2.html)
- [Linux tty(4) Manual](https://man7.org/linux/man-pages/man4/tty.4.html)
- [Linux Kernel TTY Documentation](https://docs.kernel.org/driver-api/tty/)
- [Yocto Project Documentation](https://docs.yoctoproject.org/6.0/)
