---
title: "01: STM32F407 Bare-Metal Development - From MCU Architecture to GPIO Blink"
date: "2026-11-14T00:00:00+09:00"
lastmod: "2026-11-15T00:00:00+09:00"
draft: false
description: "Set up an STM32F407 bare-metal toolchain with STM32CubeMX, CMake, ARM GCC, and OpenOCD, then flash and verify a GPIO LED blink application."
categories: ["project", "STM32-x-Firmware"]
tags: ["STM32", "Firmware", "GPIO", "Bare Metal", "STM32CubeMX", "CMake", "OpenOCD"]
weight: 1
ShowToc: true
---

**Environment:** Arch Linux (Omarchy), STM32CubeMX, CMake, ARM GCC, OpenOCD, ST-LINK  
**Hardware:** STM32F407VETx, STM32F EduBoard V1.2

## 1. Introduction

This project aims to build a sensor data acquisition and transmission system using STM32F407 and FreeRTOS.

The final system will use multiple FreeRTOS tasks, queues, interrupts, and UART communication. Before introducing an RTOS, I decided to establish a bare-metal firmware development environment and understand how the MCU interacts with its peripherals.

In this session, I studied the STM32 memory map, clock system, GPIO, and NVIC, then configured the ARM cross-compilation toolchain and successfully flashed a GPIO blink application.

## 2. STM32F407 Architecture

### 2.1 Memory-Mapped I/O

The STM32F407 uses memory-mapped I/O (MMIO), meaning peripheral registers are mapped into the CPU address space.

For example:

| Address | Region |
|---|---|
| `0x08000000` | Main Flash |
| `0x20000000` | SRAM |
| `0x10000000` | CCM RAM |
| `0x40020C00` | GPIOD registers |
| `0x40023800` | RCC registers |

Unlike ordinary memory, writing to peripheral registers can change the state of physical hardware.

```c
#define GPIOD_BASE 0x40020C00UL

volatile uint32_t *moder =
    (volatile uint32_t *)(GPIOD_BASE + 0x00);
```

The `volatile` qualifier prevents the compiler from treating these accesses as ordinary optimizable memory operations.

This is conceptually similar to MMIO access in Linux device drivers, although the address translation and access APIs differ.

### 2.2 Clock System and RCC

RCC stands for Reset and Clock Control.

It manages oscillator selection, PLL configuration, peripheral clock gating, and reset control.

Before using GPIOD, its peripheral clock must be enabled:

```c
RCC->AHB1ENR |= (1U << 3);
```

Here, bit 3 is `GPIODEN`, which enables the GPIOD clock.

The STM32F407 supports a maximum CPU clock frequency of 168 MHz, but not every peripheral necessarily operates at that frequency. AHB and APB prescalers determine the clocks supplied to different peripherals.

### 2.3 GPIO Registers

GPIO pins support several operating modes.

| Mode bits | GPIO mode |
|---|---|
| `00` | Input |
| `01` | General-purpose output |
| `10` | Alternate function |
| `11` | Analog |

The `MODER` register configures each pin's mode using two bits per pin.

For example, PD12 uses bits 25:24.

```c
GPIOD->MODER &= ~(3U << 24);
GPIOD->MODER |=  (1U << 24);
```

The `BSRR` register allows individual output bits to be set or reset.

```c
GPIOD->BSRR = (1U << 12);  // Set PD12
GPIOD->BSRR = (1U << 28);  // Reset PD12
```

Unlike a read-modify-write operation on `ODR`, `BSRR` provides atomic set/reset operations for GPIO outputs.

### 2.4 NVIC and Interrupts

The Nested Vectored Interrupt Controller (NVIC) manages interrupt enablement, prioritization, and exception delivery.

When an interrupt occurs, the CPU can suspend its current execution, enter an interrupt service routine (ISR), and subsequently resume the interrupted execution.

This concept will become important when integrating FreeRTOS, especially when using interrupt-safe RTOS APIs and configuring interrupt priorities.

No interrupt-driven peripheral experiment was performed during this session.

## 3. Establishing the Development Environment

### 3.1 Toolchain

![STM32CubeMX creating a new project](image%20%281%29.png)

*Creating a new STM32CubeMX project.*

The firmware development pipeline consists of:

```text
C source code
    |
    v
ARM GCC compiler
    |
    v
Linker + linker script
    |
    v
ELF firmware
    |
    v
OpenOCD + ST-LINK
    |
    v
STM32F407 Flash
```

The development environment uses the following tools:

- STM32CubeMX for peripheral configuration and source generation
- `arm-none-eabi-gcc` for cross-compilation
- CMake and Ninja for building
- OpenOCD for flashing and debug-server access
- ST-LINK for SWD communication

![STM32CubeMX Project Manager configured for CMake and GCC](image%20%282%29.png)

*STM32CubeMX project settings with CMake selected as the toolchain and GCC as the compiler.*

### 3.2 Confirming ST-LINK Connectivity

I first tested the ST-LINK connection using OpenOCD.

```bash
openocd \
  -f interface/stlink.cfg \
  -f target/stm32f4x.cfg \
  -c "init; targets; shutdown"
```

The resulting log included:

```text
STLINK V2J46S7 (API v2)
Target voltage: 2.899735
Cortex-M4 r0p1 processor detected
target has 6 breakpoints, 4 watchpoints
Listening on port 3333 for gdb connections
```

This verified that OpenOCD could communicate with the target through SWD.

However, it did not yet verify interactive GDB debugging.

![Development desktop with STM32CubeMX pinout and ST-LINK troubleshooting notes](image%20%283%29.png)

*Development session with the STM32F407VETx pinout in STM32CubeMX.*

## 4. Troubleshooting the Build System

### 4.1 Missing CMakeLists.txt

Initially, STM32CubeMX generated `main.c` and `stm32.ioc`, but no CMake build files appeared.

I checked the project directory using:

```bash
find ~/STM32CubeMX \
  -name CMakeLists.txt \
  -print
```

The command returned no results.

After updating or completing the STM32CubeF4 firmware package installation and regenerating the project, the following files appeared:

```text
stm32/
├── CMakeLists.txt
├── CMakePresets.json
├── cmake/
├── Core/
├── Drivers/
├── startup_stm32f407xx.s
├── stm32.ioc
└── STM32F407XX_FLASH.ld
```

![Generated STM32 project files listed in the terminal](image%20%285%29.png)

*Generated CMake files, source directories, startup assembly, and linker script.*

![STM32CubeMX extracting the STM32CubeF4 firmware package](image%20%284%29.png)

*STM32CubeF4 firmware package extraction during code generation.*

The screenshot showed CubeMX downloading and extracting the STM32CubeF4 firmware package before successful generation. This suggests the previously incomplete package installation was related to the missing files, although the exact failure cause was not independently established.

### 4.2 Incorrect Compiler Selection

The next problem occurred during compilation.

The build displayed warnings such as:

```text
warning: cast from pointer to integer of different size
[-Wpointer-to-int-cast]
```

These appeared in ST's HAL DMA driver.

Inspecting the CMake cache revealed:

```text
CMAKE_C_COMPILER:FILEPATH=/usr/bin/cc
```

CMake had selected the host compiler rather than the ARM cross-compiler.

Because the host environment uses x86-64, its pointer size differs from the STM32F407's 32-bit pointer size. This explains the warnings concerning pointer-to-`uint32_t` conversions.

The solution was to configure a fresh build directory with the generated ARM toolchain file.

```bash
cmake -S . -B build-arm -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=cmake/gcc-arm-none-eabi.cmake \
  -DCMAKE_BUILD_TYPE=Debug

cmake --build build-arm
```

This time, the output showed:

```text
[21/21] Linking C executable stm32.elf
```

The memory usage report was:

| Memory | Usage | Capacity |
|---|---:|---:|
| RAM | 1,584 B | 128 KB |
| CCMRAM | 0 B | 64 KB |
| Flash | 5,668 B | 512 KB |

These values describe the firmware at the time of that successful build, not necessarily the final blink implementation.

I also verified the target architecture:

```bash
file build-arm/stm32.elf
```

Result:

```text
ELF 32-bit LSB executable, ARM, EABI5
```

This confirmed that the firmware was compiled for ARM rather than x86-64.

## 5. Flashing the STM32F407

I used OpenOCD to program the firmware.

```bash
openocd \
  -f interface/stlink.cfg \
  -f target/stm32f4x.cfg \
  -c "program build-arm/stm32.elf verify reset exit"
```

The first attempts failed, but a subsequent attempt successfully connected to the target and completed programming.

The successful log contained:

```text
** Programming Started **
** Programming Finished **
** Verify Started **
** Verified OK **
** Resetting Target **
```

OpenOCD also reported:

```text
Unable to match requested speed 2000 kHz,
using 1800 kHz
```

This was not a flashing failure. OpenOCD selected a supported SWD clock speed instead of the exact requested value.

At this point, the firmware had been written to Flash and verified.

## 6. GPIO LED Blink Experiment

### 6.1 Hardware


![STM32F407 development board with peripherals and a ribbon cable connection](image%20%286%29.jpeg)

*STM32F407 development board used for the GPIO experiment.*

The STM32F EduBoard V1.2 schematic shows an onboard RGB LED connected to GPIO pins.

For the first experiment, I used the red channel of D1, connected to PD12.

The circuit uses a connection to 3.3V through a current-limiting resistor, making the LED active-low.

- PD12 = LOW: LED on
- PD12 = HIGH: LED off

### 6.2 GPIO Configuration

I configured PD12 as a general-purpose output in STM32CubeMX.

The relevant initialization is equivalent to:

```c
__HAL_RCC_GPIOD_CLK_ENABLE();

GPIO_InitTypeDef GPIO_InitStruct = {0};

GPIO_InitStruct.Pin = GPIO_PIN_12;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;

HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);
```

The first application used HAL's GPIO and delay functions.

```c
while (1)
{
    HAL_GPIO_TogglePin(GPIOD, GPIO_PIN_12);
    HAL_Delay(500);
}
```

The expected behavior is a GPIO transition every 500 milliseconds.

### 6.3 Initial Failure and Verification

Although the firmware was successfully flashed, the LED initially did not blink.

I investigated the GPIO initialization and application loop, then used a constant-low output test to distinguish pin-control issues from timing issues.

```c
while (1)
{
    HAL_GPIO_WritePin(
        GPIOD,
        GPIO_PIN_12,
        GPIO_PIN_RESET
    );
}
```

Because the LED is active-low, this test should turn on the red LED continuously.

After the debugging steps, LED blinking was successfully confirmed on the physical board.

![STM32 development board with an illuminated LED](image%20%287%29.jpeg)

*The onboard LED illuminated during the GPIO experiment.*

The precise final source revision was not captured in the session, so the code above documents the intended configuration and diagnostic procedure rather than an independently verified final Git commit.

## 7. Lessons Learned

### Cross-compilation must be verified

Installing `arm-none-eabi-gcc` does not automatically make CMake use it.

The build configuration must explicitly select the target toolchain, and the resulting executable should be verified.

### Successful compilation does not guarantee correct firmware

The host compiler can compile portions of embedded C code while producing output unsuitable for the target MCU.

Compiler selection and output architecture are therefore essential checks.

### Flash verification and application verification are different

OpenOCD reporting `Verified OK` means the programmed Flash content was verified.

It does not prove that the firmware performs its intended behavior.

Physical observation or debugger-based inspection is still required.

### Hardware schematics matter

Understanding the LED circuit's active-low behavior was necessary to interpret the GPIO output correctly.

The same principle will apply to sensor wiring, signal voltage levels, and communication interfaces.

## 8. Next Steps

The next milestone is UART communication using the PL2303 USB-UART adapter.

Planned experiments:

1. Configure USART3 in STM32CubeMX.
2. Connect the PL2303 adapter to the EduBoard.
3. Transmit a message using UART polling.
4. Observe the serial output from Linux.
5. Compare polling and interrupt-driven UART communication in later sessions.

This will form the basis for the final FreeRTOS-based data acquisition and transmission system.

## References

- [STM32F407 Product Documentation](https://www.st.com/en/microcontrollers-microprocessors/stm32f407ve.html)
- [STM32F4 Reference Manual (RM0090)](https://www.st.com/resource/en/reference_manual/rm0090-stm32f4xx-reference-manual-stmicroelectronics.pdf)
- [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)
- STM32F EduBoard V1.2 Schematic — local board hardware reference
- [OpenOCD Documentation](https://openocd.org/doc/html/)
