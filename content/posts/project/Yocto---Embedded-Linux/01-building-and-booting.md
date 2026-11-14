# Building and Booting Custom Embedded Linux with Yocto on Raspberry Pi 4

**Environment:** Yocto Project 6.0 (Wrynose), Raspberry Pi 4 Model B (4 GB), Omarchy (Arch Linux), Ubuntu 24.04 (Distrobox)

**Topics:** Yocto, BitBake, Embedded Linux, UART, Linux Boot Process, TTY

## 1. Introduction

The objective of this project is to build a custom Embedded Linux distribution using the Yocto Project, deploy it to a Raspberry Pi 4, and communicate with the board through UART.

The long-term goal is to develop a custom C application that communicates with an external MCU using the Linux TTY subsystem.

This stage focuses on three milestones:

1. Building a minimal Linux image using Yocto.
2. Flashing the image to a microSD card and booting the Raspberry Pi.
3. Establishing a UART serial console to inspect the Linux boot process.

## 2. Development Environment

The development environment consists of two systems.

| Component | Specification |
|---|---|
| Development PC | Omarchy (Arch-based Linux) |
| Build environment | Ubuntu 24.04 via Distrobox |
| Host RAM | 15 GiB |
| Build storage | `/mnt/data` (249 GB initially available) |
| Target hardware | Raspberry Pi 4 Model B, 4 GB |
| Yocto version | 6.0 (Wrynose) |
| BSP | `meta-raspberrypi` |
| Machine | `raspberrypi4-64` |
| Image | `core-image-minimal` |
| Serial interface | PL2303 USB-to-UART adapter |
| UART configuration | 115200 baud, 8N1 |

Ubuntu was selected as the Yocto build environment because the host operating system, Omarchy, is not one of the officially tested Yocto build distributions.

Distrobox allows an Ubuntu userspace to run without replacing the existing operating system.

The Yocto build directory is located on `/mnt/data` to avoid exhausting the host system partition.

## 3. Understanding the Yocto Build Architecture

The Yocto Project provides the tools and metadata required to create a customized Linux distribution.

Four important components are:

- **BitBake:** The build engine responsible for resolving dependencies and executing build tasks.
- **Layer:** A collection of recipes, configuration files, and related metadata.
- **Recipe:** Instructions for obtaining, configuring, compiling, and packaging software.
- **Image recipe:** Metadata defining the software included in the target root filesystem.

The relationship can be summarized as follows:

```text
OpenEmbedded-Core
       +
meta-raspberrypi
       +
 Custom Layers
       |
       v
     BitBake
       |
       v
  Build Tasks
       |
       v
   Packages
       |
       v
  Linux Image
```

For this project, the following layers were configured:

```text
core          /mnt/data/yocto/layers/openembedded-core/meta
raspberrypi   /mnt/data/yocto/meta-raspberrypi
```

The `raspberrypi` layer provides board-specific metadata, while OpenEmbedded-Core supplies the fundamental recipes and infrastructure.

The eventual custom application will be placed in a separate layer named `meta-kepp`.

### 3.1 BitBake task execution

A typical software build involves tasks such as:

```text
do_fetch
    |
do_unpack
    |
do_patch
    |
do_configure
    |
do_compile
    |
do_install
    |
do_package
```

BitBake constructs a dependency graph from the metadata and executes the required tasks.

An image build additionally assembles the root filesystem and generates deployable artifacts.

## 4. Configuring Yocto for Raspberry Pi 4

The build environment was initialized using:

```bash
cd /mnt/data/yocto
source layers/openembedded-core/oe-init-build-env build
```

The BSP layer was registered using:

```bash
bitbake-layers add-layer ../meta-raspberrypi
```

The configuration was verified using:

```bash
bitbake-layers show-layers

bitbake -e core-image-minimal | grep -E \
'^(MACHINE|IMAGE_FSTYPES|BB_NUMBER_THREADS|PARALLEL_MAKE)='
```

The resulting configuration was:

```text
MACHINE="raspberrypi4-64"
BB_NUMBER_THREADS="2"
PARALLEL_MAKE="-j2"
IMAGE_FSTYPES="tar.bz2 ext3 wic.bz2 wic.bmap"
```

The machine configuration selects the Raspberry Pi 4 64-bit target.

`BB_NUMBER_THREADS` limits the number of concurrently executing BitBake tasks, while `PARALLEL_MAKE` limits parallel compilation jobs for builds that use this variable.

Both were configured conservatively because the development PC has approximately 15 GiB of physical RAM.

## 5. Building the Linux Image

The initial image was built using:

```bash
bitbake core-image-minimal
```

`core-image-minimal` creates a small Linux system containing the essential software required to boot and operate the target device.

The generated artifacts were located in:

```text
build/tmp/deploy/images/raspberrypi4-64/
```

Several files were particularly important.

| Artifact | Purpose |
|---|---|
| `.dtb` | Device Tree Blob describing hardware |
| `.rootfs.ext3` | Root filesystem image |
| `.rootfs.tar.bz2` | Compressed root filesystem archive |
| `.rootfs.wic.bz2` | Compressed partitioned disk image |
| `.rootfs.wic.bmap` | Block map used for efficient flashing |
| `.rootfs.manifest` | Package manifest |

The `.wic.bz2` artifact was selected for deployment.

Unlike a standalone root filesystem archive, the Wic image contains a disk layout with partitions suitable for writing to removable storage.

## 6. Flashing the Linux Image to microSD

The storage device was identified using:

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
```

The microSD storage device appeared as `/dev/sda`, with a capacity of approximately 58.6 GiB.

Before writing the image, its partitions were unmounted:

```bash
sudo umount /dev/sda1 /dev/sda2
```

The compressed image was then written directly to the device:

```bash
cd /mnt/data/yocto/build/tmp/deploy/images/raspberrypi4-64

bzip2 -dc core-image-minimal-raspberrypi4-64.rootfs.wic.bz2 \
  | sudo dd of=/dev/sda bs=4M status=progress conv=fsync

sync
```

**Warning:** `dd` overwrites the specified destination device. The correct block device must be verified before executing this command.

The write operation completed, transferring approximately 361 MB.

The resulting partitions were verified:

```bash
lsblk -f /dev/sda
```

Observed layout:

```text
sda
├── sda1  vfat  FAT16  boot
└── sda2  ext4         root
```

The boot partition contains files used to start the Raspberry Pi, while the root partition contains the Linux root filesystem.

## 7. Understanding the Embedded Linux Boot Process

A simplified Raspberry Pi 4 boot sequence is:

```text
Power On
    |
    v
Raspberry Pi Boot Firmware
    |
    v
Linux Kernel
    |
    v
Hardware Initialization
    |
    v
Mount Root Filesystem
    |
    v
Start PID 1 (systemd)
    |
    v
Start System Services
    |
    v
Serial Login Prompt
```

The Raspberry Pi boot firmware loads the Linux kernel and associated configuration.

The kernel initializes hardware, memory management, scheduling, and device drivers.

It then mounts the root filesystem and starts the initial userspace process.

In this image, PID 1 was systemd.

The serial boot log confirmed that systemd initialized several services and eventually reached the login target.

## 8. UART: Universal Asynchronous Receiver/Transmitter

UART is a hardware interface for asynchronous serial communication.

Unlike synchronous interfaces that use a shared clock, UART relies on both devices agreeing on transmission parameters.

A typical UART connection uses:

| Signal | Function |
|---|---|
| TX | Transmits serial data |
| RX | Receives serial data |
| GND | Common electrical reference |

The endpoints connect in a crossed configuration:

```text
Device A                     Device B

TX  -----------------------> RX
RX  <----------------------- TX
GND ------------------------ GND
```

### 8.1 UART framing

For this experiment, the UART settings were:

```text
Baud rate: 115200
Data bits: 8
Parity:    None
Stop bits: 1
Flow:      None
```

This configuration is called **115200 8N1**.

A typical 8N1 frame contains:

```text
Start | D0 D1 D2 D3 D4 D5 D6 D7 | Stop
  0   |        8 data bits       |  1
```

UART transmits the data bits least-significant-bit first.

At 115200 baud, one bit period is approximately 8.68 microseconds.

Because an 8N1 frame contains ten bit periods, the theoretical maximum transfer rate is:

```text
115200 / 10 = 11520 bytes/s
```

This excludes additional protocol overhead.

## 9. Connecting the Raspberry Pi Through UART

A PL2303 USB-to-UART adapter was used to connect the development PC to the Raspberry Pi.

The connection was:

```text
Development PC
       |
      USB
       |
       v
PL2303 USB-UART Adapter
       |
       | TX, RX, GND
       |
       v
Raspberry Pi 4
       |
       v
Yocto Linux
```

The physical connections were:

| PL2303 | Raspberry Pi |
|---|---|
| TXD | GPIO15, physical pin 10 |
| RXD | GPIO14, physical pin 8 |
| GND | Physical pin 6 |
| VCC | Not connected |

The Raspberry Pi was powered separately through USB-C.

The UART adapter was used only for data communication.

Because Raspberry Pi GPIO operates at 3.3V, the adapter's UART signal levels must be verified before connection.

### 9.1 Enabling UART

The Raspberry Pi boot configuration was modified to include:

```ini
enable_uart=1
```

The kernel command line was also configured to direct console output to the serial interface.

These settings serve different purposes:

- `enable_uart=1` enables the UART peripheral.
- `console=` selects a device for Linux kernel console output.

These changes were made directly to the boot partition for initial experimentation. A reproducible Yocto configuration should eventually incorporate them into the BSP or custom layer metadata.

### 9.2 Connecting with picocom

On the development PC, the USB-UART adapter appeared as:

```text
/dev/ttyUSB0
```

The serial terminal was opened using:

```bash
sudo picocom -b 115200 --flow n /dev/ttyUSB0
```

Picocom reported:

```text
port is       : /dev/ttyUSB0
flowcontrol   : none
baudrate is   : 115200
parity is     : none
databits are  : 8
stopbits are  : 1

Terminal ready
```

Initially, the terminal appeared to be stuck.

However, `Terminal ready` only indicates that picocom has opened the serial port successfully. It does not mean that the Raspberry Pi is booting or that communication has been established.

After the Raspberry Pi was powered on, serial boot messages appeared.

## 10. Successful Boot Verification

The Raspberry Pi successfully initialized its Linux system and displayed the following messages:

```text
[  OK  ] Started Network Time Synchronization.
[  OK  ] Reached target Network.
[  OK  ] Started User Login Management.
[  OK  ] Started Serial Getty on ttyS0.
[  OK  ] Reached target Multi-User System.
```

The final output included:

```text
OpenEmbedded nodistro.0 raspberrypi4-64 ttyS0

raspberrypi4-64 login:
```

These messages confirm several important facts.

First, the Linux kernel successfully started the userspace initialization process.

Second, systemd initialized the system sufficiently to reach the multi-user target.

Third, the serial login service was active on `ttyS0`.

Finally, the UART connection between the Raspberry Pi and the development PC was functioning correctly.

This demonstrated that a Linux system can provide an interactive terminal without requiring a display, keyboard, or network connection.

## 11. Troubleshooting and Remaining Issues

### 11.1 Root login required a password

After reaching the serial login prompt, root login requested a password.

To investigate development login behavior, the following Yocto image feature was considered:

```bitbake
EXTRA_IMAGE_FEATURES:append = " empty-root-password"
```

Changing `local.conf` does not immediately affect an image already written to microSD.

The image must be rebuilt and redeployed for the configuration change to take effect.

At the time of writing, successful passwordless root login had not yet been verified.

### 11.2 Picocom displayed no output after restarting

After later configuration changes, picocom again displayed no incoming UART messages.

A blank serial terminal does not by itself prove that Linux failed to boot.

Possible causes include incorrect serial-console configuration, UART pin assignment, or the Raspberry Pi not transmitting data to the adapter.

Since serial boot output had previously worked, the next debugging step is to compare the working boot configuration with the modified configuration.

### 11.3 FAT filesystem warning

The boot log also displayed:

```text
FAT-fs (mmcblk0p1): Volume was not properly unmounted.
Some data may be corrupt. Please run fsck.
```

This indicates that the FAT boot filesystem was not cleanly unmounted.

It did not prevent the observed successful boot, but the filesystem should be checked if the warning persists.

Proper shutdown and safe removal procedures are important when working with removable boot media.

## 12. Connecting UART to the Linux TTY Subsystem

The most important architectural observation is that the physical UART interface becomes accessible through the Linux device-driver infrastructure.

The relationship can be summarized as:

```text
Raspberry Pi UART Hardware
            |
            v
     Linux UART Driver
            |
            v
       Serial Core
            |
            v
       TTY Subsystem
            |
            v
         /dev/ttyS0
            |
            v
     Serial Console
            |
            v
     USB-UART Adapter
            |
            v
     Development PC
```

The hardware UART transmits and receives individual bits.

The Linux serial driver manages the hardware, while the TTY subsystem provides a standard interface for terminal-related operations.

The serial console uses this infrastructure to transmit kernel messages and provide terminal access.

In a future stage, a custom C program will access a UART device through system calls and the `termios` interface:

```c
open()
read()
write()
tcgetattr()
tcsetattr()
poll()
```

This will extend the project from serial-console debugging to application-level communication with an external MCU.

## 13. Conclusion

During these two days, I built and deployed a minimal Linux image using Yocto and successfully booted it on a Raspberry Pi 4.

The most important results were:

- Building a Raspberry Pi-specific Linux image using BitBake.
- Understanding the generated root filesystem, Device Tree, and Wic image artifacts.
- Flashing the image to removable storage.
- Observing Linux and systemd initialization.
- Establishing a working UART serial console using a PL2303 adapter.
- Identifying the relationship between UART hardware, the serial driver, and the Linux TTY subsystem.

The most satisfying moment was observing the kernel and systemd boot logs through a serial terminal. It demonstrated how an embedded Linux system can be brought up and debugged using a minimal hardware interface.

The next step is to resolve the root login configuration, examine the running Linux system, and begin developing a C application that communicates with an external MCU over UART.

## References

1. [Yocto Project 6.0 Documentation](https://docs.yoctoproject.org/6.0/)
2. [Yocto Project — Overview and Concepts Manual](https://docs.yoctoproject.org/6.0/overview-manual/)
3. [Yocto Project — Development Tasks Manual](https://docs.yoctoproject.org/6.0/dev-manual/)
4. [Raspberry Pi — UART Documentation](https://www.raspberrypi.com/documentation/computers/configuration.html#uart)
5. [Linux Kernel — Serial Driver Documentation](https://docs.kernel.org/driver-api/serial/driver.html)
6. [Linux Kernel — TTY Documentation](https://docs.kernel.org/driver-api/tty/index.html)
