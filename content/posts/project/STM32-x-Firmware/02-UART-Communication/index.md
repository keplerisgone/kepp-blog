---
title: "02: STM32F407 UART Communication - Polling and Interrupt-Driven Reception"
date: "2026-11-16T00:00:00+09:00"
lastmod: "2026-11-16T00:00:00+09:00"
draft: false
description: "Transmit UART messages and implement interrupt-driven receive-and-echo on STM32F407 using USART3 and STM32 HAL."
categories: ["project", "STM32-x-Firmware"]
tags: ["STM32", "Firmware", "UART", "Interrupts", "STM32CubeMX", "HAL"]
weight: 2
ShowToc: true
---

## 1. Introduction

After successfully building and flashing a GPIO blink application, the next objective was to establish serial communication between the STM32F407 and a Linux PC.

This experiment involved two stages:

1. **UART polling transmission:** Send periodic messages from the STM32 to the PC.
2. **Interrupt-driven UART reception:** Receive data asynchronously and echo it back to the PC.

These experiments establish the communication foundation for the upcoming FreeRTOS-based sensor data acquisition system.

## 2. UART Fundamentals

### 2.1 UART and USART

UART stands for *Universal Asynchronous Receiver/Transmitter*.

It allows two devices to exchange serial data without sharing a dedicated clock signal.

The STM32F407 provides USART peripherals that support both asynchronous and synchronous communication modes.

For this experiment, USART3 operates in asynchronous mode.

The connections are:

```text
STM32F407                 PL2303                 Linux PC
-----------               --------              --------
PD8 (USART3_TX) --------> RXD
PD9 (USART3_RX) <-------- TXD
GND --------------------- GND

                          USB -----------------> /dev/ttyUSB0
```

UART is full-duplex: transmission and reception can occur independently through TX and RX.

![STM32 USART3 header connected to a PL2303 USB-to-UART adapter](image%20%281%29.jpeg)

*PL2303 adapter connected to the EduBoard USART3 header for UART communication.*

The selected communication settings were:

| Parameter | Configuration |
|---|---|
| Baud rate | 115200 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |

This configuration is called **115200 8N1**.

### 2.2 UART Frame Structure

An 8N1 UART frame contains:

```text
Idle | Start | D0 D1 D2 D3 D4 D5 D6 D7 | Stop | Idle
 High   Low      8 data bits (LSB first)    High
```

Each transmitted byte requires 10 bits:

- 1 start bit
- 8 data bits
- 1 stop bit

At 115200 baud, the theoretical maximum throughput for continuous 8N1 communication is:

\[
\frac{115200}{10} = 11520\ \text{bytes/s}
\]

This does not include application-level processing time or other communication overhead.

## 3. Day 5 — UART Polling Transmission

### 3.1 STM32CubeMX Configuration

USART3 was configured with:

- Asynchronous mode
- PD8 as USART3_TX
- PD9 as USART3_RX
- 115200 baud, 8N1
- No hardware flow control

STM32CubeMX generated the USART3 initialization function:

```c
MX_USART3_UART_Init();
```

The generated `UART_HandleTypeDef` structure, named `huart3`, represents the configured UART peripheral in the HAL driver.

### 3.2 Implementing Polling Transmission

The application transmitted a message periodically:

```c
const uint8_t msg[] = "Hello from STM32!\r\n";

while (1)
{
    HAL_UART_Transmit(
        &huart3,
        (uint8_t *)msg,
        sizeof(msg) - 1,
        1000
    );

    HAL_Delay(1000);
}
```

The HAL function has the following interface:

```c
HAL_StatusTypeDef HAL_UART_Transmit(
    UART_HandleTypeDef *huart,
    const uint8_t *pData,
    uint16_t Size,
    uint32_t Timeout
);
```

Its arguments specify the UART peripheral, the data buffer, the number of bytes, and the timeout.

`sizeof(msg) - 1` excludes the string's terminating null byte.

`HAL_UART_Transmit()` uses blocking polling. It waits for UART transmit progress, subject to the specified timeout.

Meanwhile, `HAL_Delay(1000)` creates an interval between transmission attempts.

### 3.3 Linux Serial Terminal

The PL2303 adapter appeared as:

```text
/dev/ttyUSB0
```

I initially attempted to open the device using picocom:

```bash
picocom -b 115200 --flow n /dev/ttyUSB0
```

However, Linux returned:

```text
FATAL: cannot open /dev/ttyUSB0: Permission denied
```

The immediate workaround was to run picocom with elevated permissions:

```bash
sudo picocom -b 115200 --flow n /dev/ttyUSB0
```

For a permanent solution on Arch Linux, the device's group ownership can be inspected and the user granted appropriate group membership.

### 3.4 Experimental Result

The terminal successfully displayed:

```text
Terminal ready
Hello from STM32!
Hello from STM32!
Hello from STM32!
Hello from STM32!
Hello from STM32!
```

This verified that:

- USART3 transmission worked.
- The PL2303 received the UART signal.
- Linux could read the serial data.
- The selected baud rate and framing were compatible.

The Day 5 UART polling transmission objective was completed.

## 4. Day 7 — Interrupt-Driven UART Reception

The next objective was to implement asynchronous reception using the NVIC and STM32 HAL interrupt APIs.

### 4.1 Limitations of Polling

The blocking reception function is:

```c
HAL_UART_Receive(&huart3, &byte, 1, 100);
```

It waits until a byte is received or the timeout expires.

During this operation, the calling execution context remains occupied by the HAL polling routine.

This approach is simple but becomes inefficient when the CPU must respond to other events.

Interrupt-driven reception addresses this limitation by allowing the CPU to execute other code while the UART peripheral waits for incoming data.

### 4.2 Enabling USART3 Interrupts

In STM32CubeMX, I enabled:

```text
Connectivity → USART3 → NVIC Settings
USART3 global interrupt
```

The generated interrupt service routine is:

```c
void USART3_IRQHandler(void)
{
    HAL_UART_IRQHandler(&huart3);
}
```

This function is invoked through the Cortex-M exception mechanism when the USART3 interrupt is delivered by NVIC.

`HAL_UART_IRQHandler()` then processes the peripheral's interrupt conditions.

### 4.3 Starting Interrupt-Driven Reception

Instead of calling the blocking receive function, the application starts reception with:

```c
HAL_UART_Receive_IT(&huart3, &rx_byte, 1);
```

The parameters are:

| Argument | Meaning |
|---|---|
| `&huart3` | UART handle |
| `&rx_byte` | Destination buffer |
| `1` | Number of bytes to receive |

The function configures an interrupt-driven receive operation and returns without waiting for the byte to arrive.

For a one-byte reception, the HAL eventually invokes a receive-complete callback after the byte has been received.

Importantly, the reception buffer must remain valid until the asynchronous operation completes.

## 5. Understanding the HAL Callback Mechanism

### 5.1 The Callback Function

The application defines:

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART3)
    {
        received = rx_byte;
        rx_ready = 1;
    }
}
```

This function is not called explicitly from `main()`.

Instead, it is invoked by the HAL UART driver's receive-completion handling.

The overall sequence is:

```text
PC transmits a byte
         |
         v
USART3 receives data
         |
         v
USART3 interrupt request
         |
         v
NVIC dispatches interrupt
         |
         v
USART3_IRQHandler()
         |
         v
HAL_UART_IRQHandler()
         |
         v
HAL internal receive-completion handling
         |
         v
HAL_UART_RxCpltCallback()
         |
         v
received = rx_byte
rx_ready = 1
         |
         v
Main loop processes received data
```

### 5.2 Weak Functions in the HAL

The STM32 HAL normally provides a weak default implementation of the callback:

```c
__weak void HAL_UART_RxCpltCallback(
    UART_HandleTypeDef *huart)
{
    /* Default implementation */
}
```

The application can provide its own non-weak definition:

```c
void HAL_UART_RxCpltCallback(
    UART_HandleTypeDef *huart)
{
    /* Application-specific handling */
}
```

At link time, the application's strong definition takes precedence over the weak default.

This mechanism allows application-specific interrupt behavior without modifying the HAL driver.

The callback is an ordinary C function call made by HAL, but it executes in interrupt context.

## 6. Sharing Data Between the ISR and Main Loop

### 6.1 Shared Variables

The application uses:

```c
static uint8_t rx_byte;
static volatile uint8_t received;
static volatile uint8_t rx_ready;
```

Each variable has a specific purpose.

| Variable | Purpose |
|---|---|
| `rx_byte` | HAL receive buffer |
| `received` | Stores a completed byte |
| `rx_ready` | Signals that data is ready |

The receive buffer has static storage duration, allowing it to remain valid throughout asynchronous reception.

The `volatile` qualifier ensures that the compiler preserves accesses to variables that may be modified asynchronously by an interrupt.

However, `volatile` does not provide mutual exclusion.

### 6.2 Processing Received Data

The main loop was implemented as:

```c
while (1)
{
    if (rx_ready)
    {
        uint8_t byte;

        __disable_irq();
        byte = received;
        rx_ready = 0;
        __enable_irq();

        HAL_UART_Transmit(&huart3, &byte, 1, 100);

        HAL_UART_Receive_IT(&huart3, &rx_byte, 1);
    }
}
```

Its execution consists of four stages.

**Stage 1 — Check the completion flag**

```c
if (rx_ready)
```

The main loop checks whether the interrupt callback has completed reception.

This is polling of a software flag rather than blocking on the UART receive operation.

**Stage 2 — Copy the received data**

```c
__disable_irq();
byte = received;
rx_ready = 0;
__enable_irq();
```

The short critical section prevents configurable interrupts from running while the shared data is copied and the flag cleared.

In this simple standalone firmware, the code assumes interrupts were enabled before entering the critical section. A more general implementation should preserve the previous interrupt-mask state.

**Stage 3 — Transmit the received byte**

```c
HAL_UART_Transmit(&huart3, &byte, 1, 100);
```

The application sends the byte back to the PC.

**Stage 4 — Rearm reception**

```c
HAL_UART_Receive_IT(&huart3, &rx_byte, 1);
```

Because the previous one-byte reception operation has finished, another call is needed to receive the next byte.

This design avoids placing blocking UART transmission inside the receive-complete interrupt callback.

## 7. Experimental Verification

After rebuilding and flashing the firmware, I opened picocom:

```bash
sudo picocom -b 115200 --flow n /dev/ttyUSB0
```

I initially attempted to use:

```bash
picocom -b 115200 --flow n --echo 0 /dev/ttyUSB0
```

But this failed because `--echo 0` was not accepted by the installed picocom version.

The corrected command omitted that option. Picocom reported:

```text
local echo is : no
Terminal ready
```

Characters entered through the terminal subsequently appeared in the receive output.

Because local echo was disabled, the result provided evidence that the STM32 was receiving and transmitting the characters through USART3.

The observed output was not a clean one-line echo transcript, so reliable high-throughput reception was not established by this test.

Nevertheless, the basic interrupt-driven receive-and-echo path was demonstrated.

## 8. Current Design Limitations

Although the interrupt-driven implementation worked, it is not yet suitable for robust continuous communication.

### 8.1 Reception Rearming Delay

The application rearms reception only after processing and transmitting the previously received byte.

```text
Receive byte A
      |
      v
Callback sets flag
      |
      v
Main loop handles A
      |
      v
Transmit A
      |
      v
Rearm reception
```

New bytes may arrive while the main loop is processing the previous one.

At 115200 baud and 8N1 framing, the interval between continuously transmitted bytes can be approximately 86.8 microseconds.

If the application cannot receive data quickly enough, data loss or a UART overrun may occur.

### 8.2 Interrupts Do Not Eliminate All Polling

Although the UART receive operation is interrupt-driven, the main loop still checks:

```c
if (rx_ready)
```

This is a form of software polling.

A more advanced design could use a ring buffer, interrupt-driven signaling, or an RTOS synchronization mechanism to avoid repeatedly checking the flag.

### 8.3 UART Error Handling

The current application does not explicitly handle failures such as:

- Receive overrun
- Framing errors
- UART transmit timeout
- Failed attempts to restart interrupt reception

These are important future verification targets.

## 9. Polling vs. Interrupt-Driven UART

| Property | Polling | Interrupt-driven |
|---|---|---|
| Reception API | `HAL_UART_Receive()` | `HAL_UART_Receive_IT()` |
| Execution | Blocking wait | Asynchronous receive operation |
| CPU behavior | Checks peripheral status | Runs other code until interrupt |
| Application notification | Function return | Completion callback |
| Complexity | Low | Moderate |
| Error handling | Return status | State and callback handling |
| Suitable for | Simple experiments | Event-driven communication |

Interrupt-driven reception improves responsiveness, but reliable continuous communication also requires suitable buffering and error handling.

## 10. Relationship to FreeRTOS

The final project will use FreeRTOS tasks and queues to separate sensor acquisition from data transmission.

A possible architecture is:

```text
Sensor / Peripheral
        |
        v
Interrupt / Callback
        |
        v
Data acquisition task
        |
        v
FreeRTOS Queue
        |
        v
UART output task
        |
        v
PC serial terminal
```

In an RTOS environment, interrupt handlers must use APIs specifically designed for interrupt context.

For example, FreeRTOS provides `xQueueSendFromISR()` for sending data to a queue from an ISR, subject to its interrupt-priority rules.

The UART callback experiment provides a foundation for understanding this separation between interrupt handlers and ordinary tasks.

## 11. Key Takeaways

**1. UART communication depends on matching configurations.**

The transmitter and receiver must use compatible baud rates, frame formats, voltage levels, and wiring.

**2. Polling and interrupts are different mechanisms for detecting events.**

Polling repeatedly checks state, while interrupt-driven reception allows the CPU to respond when the peripheral signals an event.

**3. HAL callbacks are invoked by the driver.**

The application does not directly call `HAL_UART_RxCpltCallback()`.

Instead, the interrupt handler calls HAL, and the HAL reception-completion path invokes the callback.

**4. Interrupt handlers should remain short.**

Lengthy blocking work inside an ISR increases interrupt latency and complicates real-time behavior.

**5. Interrupt-driven reception alone does not guarantee reliable throughput.**

Buffering, timely rearming, synchronization, and error handling are still necessary.

## 12. Next Steps

The November 16 baud-rate/framing verification will be combined with the November 18 timer-interrupt experiment.

The next experiments will cover:

- UART baud-rate mismatch and framing behavior
- TIM2 periodic interrupts
- Timer prescaler and auto-reload register calculations
- Timer interrupt callbacks
- Timing differences between interrupt-driven execution and `HAL_Delay()`

Later, these concepts will be integrated into the FreeRTOS-based sensor communication system.

## References

- [STM32F407 Product Documentation](https://www.st.com/en/microcontrollers-microprocessors/stm32f407ve.html)
- [STM32F4 Reference Manual — RM0090](https://www.st.com/resource/en/reference_manual/rm0090-stm32f4xx-reference-manual-stmicroelectronics.pdf)
- [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)
- [FreeRTOS Documentation](https://www.freertos.org/Documentation/RTOS_book.html)
- [OpenOCD Documentation](https://openocd.org/doc/html/)
- STM32F EduBoard V1.2 Schematic
