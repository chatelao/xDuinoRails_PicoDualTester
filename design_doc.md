# Design Document: In-Circuit Remote Test Environment

This document outlines the design for a hardware and software system to run an "in-circuit" remote test environment. The system will use a Raspberry Pi as a controller and two Raspberry Pi Picos as devices under test (DUTs).

## 1. High-Level Design

The system consists of three main components:

*   **Raspberry Pi 4 (Controller):**  Acts as the central controller, responsible for:
    *   Programming the Raspberry Pi Picos.
    *   Initiating and controlling test sequences.
    *   Collecting and analyzing test data.
    *   Providing a user interface for remote access.
*   **Two Raspberry Pi Picos (DUTs):** These are the devices under test. They will be configured in different modes to test various scenarios.
*   **Pi Hat:** A custom PCB that connects the Raspberry Pi and the two Picos, providing a clean and robust hardware interface.

The system will support the following testing modes:

*   **Pair Communication:** The two Picos communicate with each other to test communication protocols (e.g., UART, SPI, I2C).
*   **Observer Mode:** One Pico observes the other Pico's behavior, monitoring its GPIO pins and communication lines.
*   **Self-Observation Mode:** Each Pico monitors its own internal state and reports back to the Raspberry Pi.

## 2. Hardware Design

### 2.1. Wiring Diagram

The following diagram illustrates the wiring between the Raspberry Pi and the two Picos. The Raspberry Pi 4 acts as the central controller, connecting to both Picos for control and programming, while the Picos also have direct connections for inter-device communication tests.

```
+-----------------+
| Raspberry Pi 4  |
| (Controller)    |
+-----------------+
|       |         |
| UART0 |         | GPIO (for SWD Programming)
| SPI0  |         +--------------------------------------+
| I2C1  |         |                                      |
|       |         |                                      |
+-------+---------+                                      |
        |                                                |
        |                                                |
+-------+------------------------------------------+     |
|       |                                          |     |
|   +---+------------------+                   +---+------------------+
|   | Raspberry Pi Pico 1  |                   | Raspberry Pi Pico 2  |
+-->| (DUT 1)              |                   | (DUT 2)              |
    +----------------------+                   +----------------------+
    | SWD Port             |<------------------+ SWD Port             |
    | UART1, SPI1, I2C0    |                   | UART1, SPI1, I2C0    |
    | (for inter-Pico    )|<----------------->| (for inter-Pico    )|
    | communication)       |                   | communication)       |
    +----------------------+                   +----------------------+

```

### 2.2. Pi Hat Design

The Pi Hat will be a custom PCB that provides the following features:

*   A 40-pin GPIO header to connect to the Raspberry Pi.
*   Two sockets for the Raspberry Pi Picos.
*   **Programming Interface:** The hat will include circuitry to program the Picos using the Raspberry Pi's GPIO pins via the Serial Wire Debug (SWD) interface. This allows for a fully integrated programming and debugging solution without needing to physically connect/disconnect USB cables.
*   Level shifters to ensure compatibility between the 3.3V logic of the Raspberry Pi and the Picos.
*   Power management circuitry to provide stable power to the Picos.
*   LED indicators for power and status.
*   Easy access to the Pico's GPIO pins for probing and debugging.

## 3. Software Design

### 3.1. Raspberry Pi Software Stack

The Raspberry Pi will run a standard Raspberry Pi OS and the following software:

*   **Python:** The primary programming language for the test automation framework.
*   **picotool:** A command-line tool for programming the Raspberry Pi Picos.
*   **pySerial:** A Python library for serial communication with the Picos.
*   **Flask/Django:** A web framework for creating a remote user interface.
*   **pytest:** A testing framework for writing and running test cases.

### 3.2. Raspberry Pi Pico Firmware

The Picos will run custom firmware written in C/C++ or MicroPython. The firmware will be responsible for:

*   Implementing the different test modes.
*   Communicating with the Raspberry Pi to receive commands and send test data.
*   Controlling the Pico's GPIO pins and peripherals.

## 4. Software Stack in Detail

### 4.1. Raspberry Pi (Controller) Software Stack

The controller's software is responsible for orchestrating the entire test environment.

*   **Operating System:** Standard Raspberry Pi OS (formerly Raspbian), for its broad compatibility and community support.
*   **Test Automation Framework:** A custom Python 3 application will serve as the core of the test framework.
    *   **Orchestration:** `pytest` will be used to define and execute test cases. Its fixture model is ideal for setting up and tearing down hardware states.
    *   **Pico Programming:** The `openocd` tool, controlled via a Python `subprocess`, will be used to flash firmware onto the Picos via the SWD interface. This is more robust for automation than `picotool` which primarily uses the USB bootloader mode.
    *   **Communication:**
        *   `pySerial`: For communicating with the Picos over UART for control and data collection.
        *   `spidev`: For SPI communication if required by a test case.
        *   `smbus2`: For I2C communication.
*   **Remote User Interface:** A web-based UI will provide remote access.
    *   **Backend:** A lightweight `Flask` web server will expose a REST API to start tests, retrieve results, and view logs.
    *   **Frontend:** A simple HTML/CSS/JavaScript frontend will interact with the Flask API, providing a user-friendly dashboard.
    *   **Real-time Logging:** `WebSockets` (e.g., using `Flask-SocketIO`) will be used to stream test logs to the web interface in real-time.
*   **Data Storage:** Test results and logs will be stored locally.
    *   `SQLite`: For structured data like test results, pass/fail status, and timestamps.
    *   Plain text files (`.log`): For detailed, verbose logging from test runs.

### 4.2. Observing Pico (DUT) Software Stack

The firmware for the Pico, when acting as an observer or a participant, needs to be flexible and efficient.

*   **Firmware Language:** C/C++ using the official Raspberry Pi Pico SDK. This provides the best performance and direct access to hardware, which is critical for precise observation tasks. MicroPython is an alternative for less performance-critical test scenarios.
*   **Core Functionality:**
    *   **Command Parser:** A simple serial command parser to receive instructions from the Raspberry Pi controller (e.g., "start_test_A", "read_gpio_5", "report_status").
    *   **State Machine:** To manage the different test modes (e.g., idle, pair communication, observer, self-observation).
*   **Observer Mode Firmware:**
    *   **Logic Analyzer Functionality:** The Pico can be programmed to act as a simple logic analyzer. The `PIO` (Programmable I/O) state machines are perfect for this. A PIO program can be written to timestamp and record state changes on the GPIOs of the other DUT.
    *   **Protocol Sniffing:** For observing communication protocols like UART, SPI, or I2C, the Pico's hardware peripherals can be configured in a "sniffer" or "listen-only" mode. The captured data can be buffered and then streamed back to the controller via its own dedicated UART connection.
    *   **Analog Measurement:** The ADC can be used to monitor voltage levels on the other DUT's pins, which is useful for checking power stability or simple analog signals.
*   **Communication Protocol:** Communication with the controller will be over a UART serial connection. A simple, custom ASCII-based protocol will be used (e.g., `CMD:VALUE\n`). For larger data transfers, a more structured format like JSON or a simple binary protocol could be used.

## 5. Testing Modes

### 5.1. Pair Communication

In this mode, the two Picos will communicate with each other using a variety of protocols. The Raspberry Pi will initiate the communication and monitor the data transfer to ensure that it is correct.

### 5.2. Observer Mode

In this mode, one Pico will act as an observer, monitoring the other Pico's GPIO pins and communication lines. This is useful for debugging and for verifying that the DUT is behaving as expected.

### 5.3. Self-Observation Mode

In this mode, each Pico will monitor its own internal state, such as CPU usage, memory usage, and temperature. This information will be sent back to the Raspberry Pi for analysis.

## 6. Bill of Materials (BOM)

| Item                  | Quantity |
| --------------------- | -------- |
| Raspberry Pi 4        | 1        |
| Raspberry Pi Pico     | 2        |
| Custom Pi Hat         | 1        |
| 40-pin GPIO Header    | 1        |
| Pico Sockets          | 2        |
| Level Shifters        | 4        |
| Power Management IC   | 1        |
| LEDs                  | 4        |
| Resistors             | 8        |
| Capacitors            | 4        |

## 7. Future Improvements

*   **Wireless Communication:** Add support for wireless communication between the Picos (e.g., Bluetooth, Wi-Fi).
*   **Power Measurement:** Add circuitry to the Pi Hat to measure the power consumption of the Picos.
*   **Automated Test Generation:** Use a tool like `hypothesis` to automatically generate test cases.
*   **Cloud Integration:** Integrate the test environment with a cloud platform for data storage and analysis.

This design document provides a starting point for building the in-circuit remote test environment. The next step is to create a detailed schematic and PCB layout for the Pi Hat and to begin developing the software for the Raspberry Pi and the Picos.
