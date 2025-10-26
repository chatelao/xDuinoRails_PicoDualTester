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

The Pi Hat will be a custom PCB that provides the following features to create a flexible and powerful testing platform:

*   **Core Connectivity:**
    *   A 40-pin GPIO header to connect to the Raspberry Pi.
    *   Two sockets for the Raspberry Pi Picos.
    *   **Programming Interface:** The hat will include circuitry to program the Picos using the Raspberry Pi's GPIO pins via the Serial Wire Debug (SWD) interface. This allows for a fully integrated programming and debugging solution.
    *   Power management circuitry to provide stable power to the Picos.

*   **Advanced Test & Debug Features:**
    *   **Programmable Signal Routing:** Analog multiplexers (e.g., `74HC4053`) will be placed on the UART and SPI lines connecting the two Picos. These will be controlled by the Raspberry Pi, allowing the Picos to be programmatically isolated from each other for individual testing.
    *   **Power Monitoring:** An `INA219` current-sensing IC will be included on the power rail for each Pico. These sensors will connect to the Raspberry Pi's I2C control bus, enabling real-time power consumption measurement, which is critical for verifying low-power modes and detecting fault conditions.
    *   **Debugging Headers & Test Points:** The hat will feature 0.1" pin headers to break out the main communication buses (Pico-to-Pico UART, SPI, and I2C). Additionally, labeled test points will be provided for every GPIO pin, allowing for easy and clean connection of a logic analyzer or oscilloscope.

*   **On-Board User Interface:**
    *   **Headless Control:** Two pushbuttons will be connected to the Raspberry Pi's GPIOs to enable simple, headless control of common actions like "Reset Picos" or "Start Full Test Suite".
    *   **Status Display:** A small I2C OLED screen (e.g., SSD1306) will provide key status information without needing a dedicated monitor, displaying the Pi's IP address, the currently running test, or pass/fail results.

### 2.3. Pico-to-Pico Wiring

For direct communication tests, the two Picos are interconnected. The I2C0 peripheral on both Picos is reserved for the control bus with the Raspberry Pi, making it unavailable for Pico-to-Pico testing. All other peripherals, including both UARTs and both SPIs, are available for inter-Pico testing. For SPI, a master-slave configuration is required. To allow either Pico to act as master, a dedicated GPIO on the master is wired to the Chip Select (CSn) pin on the slave.

| Pico 1 Pin | Pico 1 Function | Pico 2 Pin | Pico 2 Function | Notes |
| :--- | :--- | :--- | :--- | :--- |
| GP0 | UART0 TX | NC | Not Connected | Reserved for RPi Data |
| GP1 | UART0 RX | NC | Not Connected | Reserved for RPi Data |
| GP2 | I2C1 SDA | GP2 | I2C1 SDA | I2C1 Bus |
| GP3 | I2C1 SCL | GP3 | I2C1 SCL | I2C1 Bus |
| GP4 | UART1 TX | GP5 | UART1 RX | UART1 Crossover |
| GP5 | UART1 RX | GP4 | UART1 TX | UART1 Crossover |
| GP6  | GPIO | GP6  | GPIO | Direct Connection |
| GP7  | GPIO | GP7  | GPIO | Direct Connection |
| GP8  | SPI1 RX (MISO) | GP8  | SPI1 RX (MISO) | SPI1 MISO |
| GP9  | SPI1 CSn | GP26 | GPIO / ADC0 | SPI1 Chip Select (Pico 2 as Master) |
| GP10 | SPI1 SCK | GP10 | SPI1 SCK | SPI1 Clock |
| GP11 | SPI1 TX (MOSI) | GP11 | SPI1 TX (MOSI) | SPI1 MOSI |
| GP12 | GPIO | GP12 | GPIO | Direct Connection |
| GP13 | GPIO | GP13 | GPIO | Direct Connection |
| GP14 | GPIO | GP14 | GPIO | Direct Connection |
| GP15 | GPIO | GP15 | GPIO | Direct Connection |
| GP16 | SPI0 RX (MISO) | GP16 | SPI0 RX (MISO) | SPI0 MISO |
| GP17 | SPI0 CSn | GP22 | GPIO | SPI0 Chip Select (Pico 2 as Master) |
| GP18 | SPI0 SCK | GP18 | SPI0 SCK | SPI0 Clock |
| GP19 | SPI0 TX (MOSI) | GP19 | SPI0 TX (MOSI) | SPI0 MOSI |
| GP20 | I2C0 SDA | NC   | Not Connected | Reserved for RPi Control Bus |
| GP21 | I2C0 SCL | NC   | Not Connected | Reserved for RPi Control Bus |
| GP22 | GPIO | GP17 | SPI0 CSn | SPI0 Chip Select (Pico 1 as Master) |
| GP23 | GPIO / SMPS PS | GP23 | GPIO / SMPS PS | Direct Connection |
| GP24 | GPIO / VBUS Sense| GP24 | GPIO / VBUS Sense| Direct Connection |
| GP25 | GPIO / LED | GP25 | GPIO / LED | Direct Connection |
| GP26 | GPIO / ADC0 | GP9  | SPI1 CSn | SPI1 Chip Select (Pico 1 as Master) |
| GP27 | GPIO / ADC1 | GP27 | GPIO / ADC1 | Direct Connection |
| GP28 | GPIO / ADC2 | GP28 | GPIO / ADC2 | Direct Connection |

### 2.4. Raspberry Pi to Picos Wiring

The Raspberry Pi controller connects to both Pico DUTs for SWD programming and I2C communication. This wiring scheme is compatible with the original 26-pin Raspberry Pi header and avoids conflicts with the Pico-to-Pico test wiring.

| Raspberry Pi Pin | Function | Pico 1 Pin | Pico 1 Function | Pico 2 Pin | Pico 2 Function | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Pico Control & Programming** | | | | | | |
| 11 (GPIO17) | GPIO | RUN | RUN | - | - | Pico 1 Reset |
| 26 (GPIO7) | GPIO | - | - | RUN | RUN | Pico 2 Reset |
| 15 (GPIO22) | GPIO | SWCLK | SWCLK | - | - | Pico 1 SWD Clock |
| 16 (GPIO23) | GPIO | SWDIO | SWDIO | - | - | Pico 1 SWD Data |
| 18 (GPIO24) | GPIO | - | - | SWCLK | SWCLK | Pico 2 SWD Clock |
| 22 (GPIO25) | GPIO | - | - | SWDIO | SWDIO | Pico 2 SWD Data |
| **Data & Control Bus** | | | | | | |
| 3 (GPIO2) | I2C1 SDA | GP20 | I2C0 SDA | GP20 | I2C0 SDA | RPi to Picos Control Bus (also for INA219, OLED) |
| 5 (GPIO3) | I2C1 SCL | GP21 | I2C0 SCL | GP21 | I2C0 SCL | RPi to Picos Control Bus (also for INA219, OLED) |
| 8 (GPIO14) | UART0 TX | GP0 | UART0 TX | - | - | RPi to Pico 1 Data Download |
| 10 (GPIO15)| UART0 RX | GP1 | UART0 RX | - | - | Pico 1 to RPi Data Download |
| **On-Hat Component Control** | | | | | | |
| 7 (GPIO4) | GPIO | - | - | - | - | Multiplexer Control 0 |
| 12 (GPIO18)| GPIO | - | - | - | - | Multiplexer Control 1 |
| 13 (GPIO27)| GPIO | - | - | - | - | Pushbutton 1 |
| 19 (GPIO10)| GPIO | - | - | - | - | Pushbutton 2 |

### 2.5. Available GPIOs for Status LEDs

After accounting for all the necessary connections for SWD, the control bus, data transfer, and on-hat components, there are still GPIO pins available on the Raspberry Pi's 26-pin header for status LEDs.

*   **Available Pins:** 21 (GPIO9), 23 (GPIO11), 24 (GPIO8).
*   **Total Available:** 3 GPIO pins.

These pins can be connected to LEDs through current-limiting resistors to provide visual feedback on the status of the test environment, such as "Test in Progress," "Test Passed," or "Test Failed."

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

#### 4.2.1. Data Download Protocol

When the observing Pico has captured a significant amount of data (e.g., from the logic analyzer or protocol sniffer), it needs an efficient way to transfer this data to the Raspberry Pi controller for analysis and storage. The download process will be initiated by the controller and will use a combination of the I2C control bus and a dedicated UART connection.

The process is as follows:

1.  **Initiate Transfer:** The Raspberry Pi sends a command to the observing Pico via the I2C control bus. This command, such as `START_DATA_DUMP`, instructs the Pico to prepare for data transmission.
2.  **Acknowledge and Prepare:** The observing Pico receives the command and acknowledges it over I2C. It then prepares the captured data, which may be stored in a buffer or a ring buffer in its RAM. The Pico will indicate the total size of the data to be transferred in its acknowledgment.
3.  **High-Speed Data Transfer:** The Raspberry Pi opens a serial connection to the observing Pico's dedicated UART interface. The Pico then begins sending the data over UART at a high baud rate (e.g., 921600 or higher). Using UART for the bulk data transfer is much more efficient than I2C for large payloads.
4.  **Flow Control:** To prevent buffer overruns on the Raspberry Pi, either hardware flow control (RTS/CTS) or a software-based acknowledgment protocol (e.g., sending an "ACK" byte after every 64 bytes) will be used.
5.  **End of Transfer:** Once all the data has been sent, the Pico sends an `END_OF_TRANSMISSION` message. The Raspberry Pi verifies the integrity of the received data, for instance, by checking a checksum (like CRC32) sent by the Pico at the end of the transmission.
6.  **Confirmation:** The Raspberry Pi sends a final `TRANSFER_COMPLETE` command over the I2C bus to confirm a successful download. The Pico can then safely clear its data buffers, ready for the next observation task.

## 5. Testing Modes

### 5.1. Pair Communication

In this mode, the two Picos will communicate with each other using a variety of protocols. The Raspberry Pi will initiate the communication and monitor the data transfer to ensure that it is correct.

### 5.2. Observer Mode

In this mode, one Pico will act as an observer, monitoring the other Pico's GPIO pins and communication lines. This is useful for debugging and for verifying that the DUT is behaving as expected.

### 5.3. Self-Observation Mode

In this mode, each Pico will monitor its own internal state, such as CPU usage, memory usage, and temperature. This information will be sent back to the Raspberry Pi for analysis.

## 6. Bill of Materials (BOM)

| Item | Quantity | Estimated Price (USD) | Notes |
| :--- | :--- | :--- | :--- |
| **Core Components** | | | |
| Raspberry Pi 4 (4GB) | 1 | $55.00 | Controller |
| Raspberry Pi Pico | 2 | $8.00 ($4.00 each) | Devices Under Test (DUTs) |
| Custom Pi Hat PCB | 1 | $20.00 | Price can vary based on manufacturer and quantity |
| **Hat Components** | | | |
| 40-pin GPIO Header | 1 | $1.50 | |
| Pico Sockets | 2 | $2.00 ($1.00 each) | |
| Analog Multiplexer (74HC4053) | 2 | $3.00 ($1.50 each) | For programmable signal routing |
| Power Monitor IC (INA219) | 2 | $8.00 ($4.00 each) | For Pico power measurement |
| 0.96" I2C OLED Screen (SSD1306) | 1 | $8.00 | For status display |
| Pushbuttons | 2 | $1.00 ($0.50 each) | For headless control |
| Debug Headers (2x10) | 2 | $2.00 ($1.00 each) | For logic analyzer access |
| Power Management IC (e.g., AP2210) | 1 | $2.00 | |
| Resistors (assorted) | 1 pack | $5.00 | Pull-ups, current-limiting, etc. |
| Capacitors (assorted) | 1 pack | $5.00 | Decoupling, filtering, etc. |
| **Total Estimated Cost** | | **$120.50** | |

## 7. Future Improvements

*   **Wireless Communication:** Add support for wireless communication between the Picos (e.g., Bluetooth, Wi-Fi).
*   **Power Measurement:** Add circuitry to the Pi Hat to measure the power consumption of the Picos.
*   **Automated Test Generation:** Use a tool like `hypothesis` to automatically generate test cases.
*   **Cloud Integration:** Integrate the test environment with a cloud platform for data storage and analysis.

This design document provides a starting point for building the in-circuit remote test environment. The next step is to create a detailed schematic and PCB layout for the Pi Hat and to begin developing the software for the Raspberry Pi and the Picos.
