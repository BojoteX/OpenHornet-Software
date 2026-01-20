# OpenHornet ABSIS RS-485 Communication Protocol Research

**Document Version:** 1.0
**Date:** 2026-01-20
**Purpose:** Technical reference for implementing an ESP32-based RS-485 master (CockpitOS) compatible with OpenHornet ABSIS slave devices.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [RS-485 Physical Layer](#3-rs-485-physical-layer)
4. [DCS-BIOS Protocol Layer](#4-dcs-bios-protocol-layer)
5. [RS-485 Bus Protocol Specification](#5-rs-485-bus-protocol-specification)
6. [Slave Device Implementation](#6-slave-device-implementation)
7. [Master Device Implementation](#7-master-device-implementation)
8. [OpenHornet Panel Inventory](#8-openhornet-panel-inventory)
9. [Implementation Considerations for ESP32](#9-implementation-considerations-for-esp32)
10. [Known Issues and Limitations](#10-known-issues-and-limitations)
11. [Resources and References](#11-resources-and-references)

---

## 1. Executive Summary

### Current State of ABSIS Software

The OpenHornet ABSIS (Arduino Based Simulator Interface System) **does have working RS-485 slave implementations** in the [OpenHornet-Software repository](https://github.com/jrsteensen/OpenHornet-Software). However:

- **Bus Master software exists** in the DCS-BIOS Arduino library
- **Slave firmware** is available for 19+ cockpit panels
- **Protocol is DCS-BIOS standard** - not a custom OpenHornet protocol
- **Hardware is production-ready** via OpenCockpit Shop vendors

### Key Findings

| Question | Answer |
|----------|--------|
| Protocol Specification | DCS-BIOS RS-485 protocol (Gadroc/DCS-Skunkworks) |
| Baud Rate | **250,000 bps** |
| Addressing | Polled, 0-127 slaves per bus (32 active recommended) |
| Message Format | Binary framed: `[Address][MsgType][DataLen][Data...][Checksum]` |
| Frame Sync | `0x55 0x55 0x55 0x55` (4-byte sequence) |
| Timeout | 2ms slave response timeout |

---

## 2. System Architecture Overview

### ABSIS Network Topology

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              PC (Windows)                                │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      DCS World Simulator                         │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │              DCS-BIOS Lua Export Script                   │  │    │
│  │  │  • Monitors cockpit state changes                         │  │    │
│  │  │  • Sends UDP multicast to 239.255.50.10:5010              │  │    │
│  │  │  • Receives control inputs via TCP/UDP                    │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                          USB Serial @ 250000 bps                         │
│                                   ▼                                      │
└───────────────────────────────────┼──────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼──────────────────────────────────────┐
│                      ABSIS Bus Master PCB                                │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     Arduino Mega 2560                            │    │
│  │  • UART0: USB to PC                                              │    │
│  │  • UART1: RS-485 Bus 1 via MAX487 (Pin 2 TXENABLE)               │    │
│  │  • UART2: RS-485 Bus 2 via MAX487 (Pin 3 TXENABLE)               │    │
│  │  • UART3: RS-485 Bus 3 via MAX487 (Pin 4 TXENABLE)               │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                    │              │              │                       │
└────────────────────┼──────────────┼──────────────┼───────────────────────┘
                     │              │              │
        ┌────────────┘              │              └────────────┐
        ▼                           ▼                           ▼
┌───────────────┐          ┌───────────────┐          ┌───────────────┐
│  RS-485 Bus 1 │          │  RS-485 Bus 2 │          │  RS-485 Bus 3 │
│  (Daisy-chain)│          │  (Daisy-chain)│          │  (Daisy-chain)│
└───────┬───────┘          └───────┬───────┘          └───────┬───────┘
        │                          │                          │
   ┌────┴────┐                ┌────┴────┐                ┌────┴────┐
   ▼         ▼                ▼         ▼                ▼         ▼
┌─────┐  ┌─────┐          ┌─────┐  ┌─────┐          ┌─────┐  ┌─────┐
│Slave│  │Slave│          │Slave│  │Slave│          │Slave│  │Slave│
│Addr1│  │Addr2│          │Addr1│  │Addr2│          │Addr1│  │Addr2│
└─────┘  └─────┘          └─────┘  └─────┘          └─────┘  └─────┘
```

### Communication Flow

```
    PC                    Bus Master              RS-485 Bus               Slave
     │                         │                       │                     │
     │  Export Data (UDP)      │                       │                     │
     │─────────────────────────>                       │                     │
     │                         │                       │                     │
     │  Serial Stream          │                       │                     │
     │─────────────────────────>                       │                     │
     │                         │                       │                     │
     │                         │  Poll Request         │                     │
     │                         │  [Addr][Type][Len]    │                     │
     │                         │───────────────────────>─────────────────────>
     │                         │  + Export Data        │                     │
     │                         │                       │                     │
     │                         │                       │  Poll Response      │
     │                         │                       │  [Addr][Type][Len]  │
     │                         │<──────────────────────<─────────────────────│
     │                         │  + Input Data         │                     │
     │                         │                       │                     │
     │  Input Commands         │                       │                     │
     │<─────────────────────────                       │                     │
     │                         │                       │                     │
```

---

## 3. RS-485 Physical Layer

### Hardware Components

#### ABSIS Bus Master PCB
- **MCU:** Arduino Mega 2560 R3 (ATmega2560)
- **Transceivers:** 3x MAX487E RS-485 ICs
- **Bus Count:** Up to 3 independent RS-485 buses
- **Power Input:** 8-pin Molex Mini-Fit Jr (12V, 5V, 3.3V, GND)

#### ABSIS Slave PCBs

| Board Type | MCU | Form Factor | Use Case |
|------------|-----|-------------|----------|
| ABSIS ALE | ATmega328P | Nano footprint | Simple panels (≤8 analog, ≤11 digital) |
| ABSIS ALE w/ Relay | ATmega328P | Nano + relay | Mag-switch panels |
| ABSIS Mega | ATmega2560 | Mega shield | Complex panels (≤51 digital, ≤16 analog) |

### Connector Pinout (6-pin Molex Mini-Fit Jr)

| Pin | Signal | Description |
|-----|--------|-------------|
| 1 | +12V | Power for backlighting, solenoids |
| 2 | +5V | MCU power |
| 3 | +3.3V | Optional 3.3V logic |
| 4 | RS485-A | Differential pair A (non-inverting) |
| 5 | RS485-B | Differential pair B (inverting) |
| 6 | GND | Common ground |

### Electrical Specifications

| Parameter | Value |
|-----------|-------|
| Signaling | Half-duplex RS-485 |
| Baud Rate | 250,000 bps |
| Cable Type | Twisted pair (A/B) + ground |
| Termination | 120Ω at bus ends (recommended) |
| Max Nodes | 32 standard / 256 with repeaters |
| Max Cable Length | ~1200m @ 100kbps, ~100m @ 250kbps |

### Transceiver Pin Connections (MAX487E)

```
Arduino Mega          MAX487E              RS-485 Bus
─────────────         ───────              ──────────
    TX  ──────────────> DI
    RX  <────────────── RO
TXENABLE ─────────────> RE (active LOW)
TXENABLE ─────────────> DE (active HIGH)
                        A  ──────────────────> A (Bus)
                        B  ──────────────────> B (Bus)
                       GND ──────────────────> GND
                       VCC ──────────────────> +5V
```

### TXENABLE Pin Assignments

| UART | Arduino Pin | DCS-BIOS Define |
|------|-------------|-----------------|
| UART0 | (USB only) | N/A |
| UART1 | Pin 2 | `UART1_TXENABLE_PIN 2` |
| UART2 | Pin 3 | `UART2_TXENABLE_PIN 3` |
| UART3 | Pin 4 | `UART3_TXENABLE_PIN 4` |
| Slave | Pin 5 | `TXENABLE_PIN 5` |

---

## 4. DCS-BIOS Protocol Layer

### Export Stream Format (PC → Bus Master)

DCS-BIOS exports cockpit state via a binary stream. The Bus Master receives this via USB serial and relays it to slaves.

#### Frame Synchronization

Every update frame begins with a 4-byte sync sequence:

```
0x55 0x55 0x55 0x55
```

**Properties:**
- Sent at ~30 Hz (30 updates/second)
- Cannot appear in normal data (protocol guarantees this)
- Used by receivers to synchronize after errors

#### Write Access Format

```
┌─────────────────┬─────────────────┬─────────────────────────────┐
│  Start Address  │   Data Length   │         Data Bytes          │
│    (2 bytes)    │    (2 bytes)    │      (N bytes, N=len)       │
│   little-endian │   little-endian │       little-endian         │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

**Example Write:**
```
Address: 0x1000
Length:  0x0004 (4 bytes)
Data:    0x41 0x2D 0x31 0x30 ("A-10" ASCII)

Wire bytes: 0x00 0x10 0x04 0x00 0x41 0x2D 0x31 0x30
```

#### Address Space Layout

| Address Range | Module | Description |
|---------------|--------|-------------|
| 0x0000-0x0400 | MetadataStart | Always active, aircraft ID |
| 0x0400-0x0800 | CommonData | Generic flight data |
| 0x0800+ | Aircraft-specific | Per-aircraft controls/displays |
| 0x5555 | End-of-Update | Special sync marker |

### Integer Value Encoding

Cockpit values are packed into 16-bit words with bitmasks:

```c
// Example: Master Arm Switch at address 0x740C, mask 0x2000, shift 13
uint16_t raw_word = *(uint16_t*)(&exportData[0x740C]);
uint16_t value = (raw_word & 0x2000) >> 13;  // 0 or 1
```

### Input Command Format (Slave → PC)

Slaves send text-based commands:

```
<CONTROL_NAME> <VALUE>\n
```

**Examples:**
```
MASTER_ARM_SW 1\n
FLIR_SW 2\n
INS_SW 5\n
```

---

## 5. RS-485 Bus Protocol Specification

### Protocol Overview

The DCS-BIOS RS-485 protocol implements **master-initiated polling** with broadcast data distribution:

1. **Master polls slaves sequentially** (round-robin)
2. **Export data is broadcast** to all slaves in poll packets
3. **Only addressed slave responds** with input changes
4. **2ms timeout** before master moves to next slave

### Packet Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Byte 0          │ Byte 1        │ Byte 2       │ Bytes 3..N  │ Final    │
│ [Type:3][Addr:5]│ Data Length   │ Data...      │ ...         │ Checksum │
├─────────────────┼───────────────┼──────────────┼─────────────┼──────────┤
│ TTT AAAAA       │ 0x00-0xFF     │ Payload      │ Payload     │ XOR/CRC  │
└─────────────────┴───────────────┴──────────────┴─────────────┴──────────┘

TTT = Packet Type (3 bits, values 0-7)
AAAAA = Slave Address (5 bits, values 0-31)
```

### Packet Types

| Type ID | Name | Direction | Description |
|---------|------|-----------|-------------|
| 0 | Polling Request | Master→Slave | Regular poll + export data |
| 1 | Polling Response | Slave→Master | Input data (switch states) |
| 2 | Config Request | Master→Slave | Request device configuration |
| 3 | Config Response | Slave→Master | Device capabilities/info |
| 4-7 | Reserved | — | Future use |

### Addressing

| Address Range | Usage |
|---------------|-------|
| 0 | Reserved (triggers new device scan) |
| 1-31 | Active polled devices |
| 32-126 | Extended range (less frequently polled) |
| 127 | Broadcast (listen-only devices) |

### Timing Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Baud Rate | 250,000 bps | 4µs per bit |
| Response Timeout | 2 ms | Master waits before moving on |
| Sync Silence | 500 µs | Required bus silence for sync |
| RX Timeout (data) | 5 ms | Timeout waiting for packet data |
| RX Timeout (length) | 1 ms | Timeout for non-existent device |
| Poll Rate | ~3 Hz per slave | 32 slaves = ~10ms cycle |

### State Machine (Slave)

```
                    ┌─────────────────┐
                    │  UNINITIALIZED  │
                    └────────┬────────┘
                             │ DcsBios::setup()
                             ▼
                    ┌─────────────────┐
          ┌────────>│      SYNC       │<────────┐
          │         │ (wait 500µs     │         │
          │         │  bus silence)   │         │
          │         └────────┬────────┘         │
          │                  │ silence detected │
          │                  ▼                  │
          │         ┌─────────────────┐         │
          │         │ RX_WAIT_ADDRESS │         │
          │         └────────┬────────┘         │
          │                  │ byte received    │
          │ timeout          ▼                  │
          │         ┌─────────────────┐         │
          │         │ RX_WAIT_MSGTYPE │         │
          │         └────────┬────────┘         │
          │                  │                  │
          │                  ▼                  │
          │         ┌─────────────────┐         │
          │         │RX_WAIT_DATALENGTH         │
          │         └────────┬────────┘         │
          │                  │                  │
          │                  ▼                  │
          │         ┌─────────────────┐         │
          │         │   RX_WAIT_DATA  │<──┐     │
          │         └────────┬────────┘   │     │
          │                  │ more data  │     │
          │                  └────────────┘     │
          │                  │ all received     │
          │                  ▼                  │
          │         ┌─────────────────┐         │
          │         │ RX_WAIT_CHECKSUM│         │
          │         └────────┬────────┘         │
          │                  │                  │
          │                  ▼                  │
          │         ┌─────────────────┐         │
          │         │RX_HOST_MSG_DONE │         │
          │         └────────┬────────┘         │
          │                  │                  │
          │   if addressed   │   broadcast      │
          │         ┌────────┴────────┐         │
          │         ▼                 ▼         │
          │  ┌────────────┐    ┌────────────┐   │
          │  │ TX_RESPOND │    │  (ignore)  │───┘
          │  └─────┬──────┘    └────────────┘
          │        │
          │        ▼
          │  ┌────────────┐
          │  │TX_SEND_DATA│
          │  └─────┬──────┘
          │        │
          │        ▼
          │  ┌────────────┐
          └──│ TX_CHECKSUM│
             └────────────┘
```

### Checksum Implementation

**Current State:** The DCS-BIOS library has incomplete checksum implementation:
- Slave TX hardcodes `0x72` as checksum
- Slave RX ignores incoming checksum
- Comment: "TODO: check checksum here"

**Recommended Implementation:**
```c
// XOR checksum over all bytes except checksum itself
uint8_t calculateChecksum(uint8_t* data, size_t length) {
    uint8_t checksum = 0;
    for (size_t i = 0; i < length; i++) {
        checksum ^= data[i];
    }
    return checksum;
}
```

---

## 6. Slave Device Implementation

### Required Defines

```c
// Option 1: RS-485 Slave Mode
#define DCSBIOS_RS485_SLAVE <address>  // 1-126

// Option 2: Direct USB (no RS-485)
#define DCSBIOS_IRQ_SERIAL   // ATmega328P/2560 only
// or
#define DCSBIOS_DEFAULT_SERIAL  // Other MCUs

// RS-485 Transceiver Control
#define TXENABLE_PIN 5

// UART Selection (Mega only)
#define UART1_SELECT  // or UART0_SELECT
```

### Minimal Slave Sketch

```c
#define DCSBIOS_RS485_SLAVE 1
#define TXENABLE_PIN 5

#if defined(__AVR_ATmega328P__) || defined(__AVR_ATmega2560__)
#define DCSBIOS_IRQ_SERIAL
#else
#define DCSBIOS_DEFAULT_SERIAL
#endif

#include "DcsBios.h"

// Define switch connected to pin 3
DcsBios::Switch2Pos masterArmSw("MASTER_ARM_SW", 3);

void setup() {
    DcsBios::setup();
}

void loop() {
    DcsBios::loop();
}
```

### DCS-BIOS Input Classes

| Class | Description | Example |
|-------|-------------|---------|
| `Switch2Pos` | 2-position switch | `Switch2Pos sw("CTRL", pin)` |
| `Switch3Pos` | 3-position switch | `Switch3Pos sw("CTRL", pinA, pinB)` |
| `SwitchMultiPos` | Rotary switch | `SwitchMultiPos sw("CTRL", pins[], count)` |
| `Potentiometer` | Analog input | `Potentiometer pot("CTRL", pin)` |
| `RotaryEncoder` | Encoder knob | `RotaryEncoder enc("CTRL", pinA, pinB)` |
| `ActionButton` | Momentary button | `ActionButton btn("CTRL", pin, "CMD")` |

### DCS-BIOS Output Classes

```c
// Receive cockpit state updates
void onMasterArmChange(unsigned int newValue) {
    // newValue contains decoded state (0 or 1 for this control)
    digitalWrite(LED_PIN, newValue);
}
DcsBios::IntegerBuffer masterArmBuffer(0x740C, 0x2000, 13, onMasterArmChange);

// String output (e.g., UFC display)
void onUfcScratchpadChange(char* newValue) {
    lcd.print(newValue);
}
DcsBios::StringBuffer<8> ufcScratchpad(0x744E, onUfcScratchpadChange);
```

### Manual Message Transmission

```c
// Send command when DCS-BIOS classes aren't suitable
char msg[] = "MASTER_ARM_SW";
char arg[] = "1";
if (DcsBios::tryToSendDcsBiosMessage(msg, arg)) {
    // Message queued successfully
}
// Returns false if ring buffer full - retry later
```

---

## 7. Master Device Implementation

### Bus Master Configuration

```c
#define DCSBIOS_RS485_MASTER

// UART TXENABLE pins (comment out unused)
#define UART1_TXENABLE_PIN 2
#define UART2_TXENABLE_PIN 3
#define UART3_TXENABLE_PIN 4

#include "DcsBios.h"

void setup() {
    DcsBios::setup();
}

void loop() {
    DcsBios::loop();
}
```

### Master Responsibilities

1. **Receive export stream** from PC via USB Serial (UART0)
2. **Parse DCS-BIOS frames** (sync detection, address/data extraction)
3. **Distribute export data** to all buses via polling
4. **Poll slaves sequentially** (round-robin across addresses 1-127)
5. **Handle slave responses** (input commands)
6. **Forward input commands** to PC via USB Serial
7. **Device discovery** (scan for new slaves at address 0)

### Polling Algorithm

```c
// Simplified master polling logic
void masterPollCycle() {
    static uint8_t pollAddress = 1;
    static uint8_t scanAddress = 1;
    static bool slavePresent[128] = {false};

    // Poll address 0 triggers device scan
    if (pollAddress == 0) {
        // Scan for new device at scanAddress
        pollSlave(scanAddress);
        scanAddress = (scanAddress % 127) + 1;
        pollAddress = 1;
    } else {
        // Poll known slaves
        if (slavePresent[pollAddress]) {
            pollSlave(pollAddress);
        }
        pollAddress = (pollAddress + 1) % 128;
    }
}

void pollSlave(uint8_t address) {
    // Build poll request packet
    uint8_t packet[MAX_PACKET_SIZE];
    packet[0] = (0x00 << 5) | (address & 0x1F);  // Type 0, Address
    packet[1] = exportDataLength;
    memcpy(&packet[2], exportData, exportDataLength);
    packet[2 + exportDataLength] = calculateChecksum(packet, 2 + exportDataLength);

    // Transmit
    setTxEnable(true);
    serial.write(packet, 3 + exportDataLength);
    serial.flush();
    setTxEnable(false);

    // Wait for response (2ms timeout)
    unsigned long start = micros();
    while (micros() - start < 2000) {
        if (serial.available()) {
            // Parse response
            handleSlaveResponse();
            return;
        }
    }
    // Timeout - slave not present
}
```

---

## 8. OpenHornet Panel Inventory

### Implemented RS-485 Slave Panels

| Panel | Ref Des | Address | Board Type | Status |
|-------|---------|---------|------------|--------|
| Master Arm Panel | 1A2 | 1 | ABSIS ALE | Implemented |
| HUD Panel | 1A7 | 1 | ABSIS ALE | Implemented |
| Spin/Recovery Panel | 1A6 | 1 | ABSIS ALE | Implemented |
| L DDI & EWI | 1A3 | 2 | ABSIS ALE | In Development |
| Seat Controls | 3A2A1 | 9 | ABSIS ALE | Implemented |
| Ext Lights Panel | 4A4A2 | 4 | ABSIS ALE | Implemented |
| APU Panel | 4A5A2 | 5 | ABSIS ALE w/ Relay | Implemented |
| Fuel Panel | 4A5A1 | 6 | ABSIS ALE w/ Relay | Implemented |
| FCS Panel | 4A6A1 | 7 | ABSIS ALE | Implemented |
| Comm Panel | 4A7A1 | 8 | ABSIS Mega | In Development |
| OBOGS Panel | 4A7A2 | 10 | ABSIS ALE | Implemented |
| Select Jettison | 4A3A1 | 1 | ABSIS ALE w/ Relay | In Development |
| Landing Gear Panel | 4A2A1 | — | ABSIS ALE w/ Relay | In Development |
| Interior Lights Panel | 5A6A1 | 5 | ABSIS ALE | Implemented |
| Sensor Panel | 5A7A1 | 6 | ABSIS ALE w/ Relay | Implemented |
| SIM Control Panel | 5A8A1 | 7 | ABSIS ALE | Implemented |
| KY-58 Panel | 5A9A1 | 8 | ABSIS ALE | Implemented |
| Defog Panel | 5A10 | 9 | ABSIS ALE w/ Relay | Implemented |

### Bus Address Allocation

**Note:** Addresses must be unique **per bus**, not globally. Each of the 3 RS-485 buses can have slaves with addresses 1-127.

Typical allocation:
- **Bus 1:** Upper Instrument Panel (OH1 prefix)
- **Bus 2:** Left Console (OH4 prefix)
- **Bus 3:** Right Console (OH5 prefix) + Center Tub (OH3)

---

## 9. Implementation Considerations for ESP32

### Recommended Hardware

| Component | Specification |
|-----------|---------------|
| MCU | ESP32-WROOM-32 or ESP32-S3 |
| RS-485 Transceiver | MAX485 or MAX3485 (3.3V compatible) |
| UART | Hardware UART (UART1 or UART2) |
| Level Shifter | Required if using 5V transceiver |

### ESP32 UART Configuration

```c
#include <HardwareSerial.h>

#define RS485_UART_NUM 1
#define RS485_TX_PIN 17
#define RS485_RX_PIN 16
#define RS485_TXENABLE_PIN 4
#define RS485_BAUD 250000

HardwareSerial RS485Serial(RS485_UART_NUM);

void setup() {
    RS485Serial.begin(RS485_BAUD, SERIAL_8N1, RS485_RX_PIN, RS485_TX_PIN);
    pinMode(RS485_TXENABLE_PIN, OUTPUT);
    digitalWrite(RS485_TXENABLE_PIN, LOW);  // Receive mode
}

void setTxEnable(bool enable) {
    digitalWrite(RS485_TXENABLE_PIN, enable ? HIGH : LOW);
    if (!enable) {
        delayMicroseconds(10);  // Allow transceiver to switch
    }
}
```

### WiFi Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            ESP32 CockpitOS                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        WiFi Module                               │   │
│  │  • UDP listener on 239.255.50.10:5010 (DCS-BIOS export)          │   │
│  │  • TCP client for control input to DCS-BIOS                      │   │
│  │  • Optional: Web UI for configuration                            │   │
│  └───────────────────────────────┬─────────────────────────────────┘   │
│                                  │                                      │
│  ┌───────────────────────────────┴─────────────────────────────────┐   │
│  │                    Protocol Translator                           │   │
│  │  • Parse DCS-BIOS export frames                                  │   │
│  │  • Build RS-485 poll packets                                     │   │
│  │  • Parse slave responses                                         │   │
│  │  • Format input commands for DCS-BIOS                            │   │
│  └───────────────────────────────┬─────────────────────────────────┘   │
│                                  │                                      │
│  ┌───────────────────────────────┴─────────────────────────────────┐   │
│  │                    RS-485 Master Engine                          │   │
│  │  • Multi-bus support (up to 3 UARTs)                             │   │
│  │  • Slave polling scheduler                                       │   │
│  │  • TXENABLE control                                              │   │
│  │  • Response timeout handling                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Implementation Notes

1. **Baud Rate Accuracy:** ESP32 can achieve 250,000 bps accurately with default clock
2. **Interrupt Handling:** Use UART interrupts or FreeRTOS tasks for non-blocking operation
3. **Timing:** Use `micros()` for sub-millisecond timing; avoid `delay()` in polling loop
4. **Buffer Management:** Implement ring buffers for both export data and input commands
5. **Multi-tasking:** Consider FreeRTOS tasks for WiFi/RS-485/UI separation

### Compatibility Considerations

| Aspect | Arduino Mega (Reference) | ESP32 (Target) |
|--------|--------------------------|----------------|
| Voltage | 5V logic | 3.3V logic |
| UARTs | 4 hardware | 3 hardware |
| Clock | 16 MHz | 240 MHz |
| RAM | 8 KB | 520 KB |
| WiFi | External shield | Built-in |
| Multi-core | No | Yes (dual-core) |

---

## 10. Known Issues and Limitations

### DCS-BIOS Library Issues

1. **Pro Micro Incompatibility**
   - 32U4-based boards fail to compile RS-485 mode
   - Workaround: Use ATmega328P or ATmega2560

2. **Incomplete Checksum**
   - TX: Hardcoded `0x72` value
   - RX: Checksum not validated
   - Impact: Low (bus errors are rare on short runs)

3. **Pull Request #56 Pending**
   - Enhanced debouncing for `SwitchMultiPos`
   - OpenHornet implements custom `SwitchMultiPosDebounce` class

### Protocol Limitations

1. **Polling Overhead**
   - Each slave polled ~3x/second
   - 32 slaves = ~100ms total cycle time
   - Latency varies with slave count

2. **No Slave-Initiated Transmission**
   - Slaves cannot transmit without being polled
   - Input changes wait until next poll

3. **Single-Bus Bandwidth**
   - All export data relayed through each poll
   - Large aircraft may exceed packet limits

### Hardware Considerations

1. **Bus Length vs. Speed**
   - 250 kbps limits practical cable length
   - Use twisted pair with proper termination

2. **Ground Loops**
   - Multiple power supplies can cause issues
   - Ensure common ground reference

---

## 11. Resources and References

### Official Repositories

- **OpenHornet Software:** [github.com/jrsteensen/OpenHornet-Software](https://github.com/jrsteensen/OpenHornet-Software)
- **OpenHornet Hardware:** [github.com/jrsteensen/OpenHornet](https://github.com/jrsteensen/OpenHornet)
- **DCS-BIOS:** [github.com/DCS-Skunkworks/dcs-bios](https://github.com/DCS-Skunkworks/dcs-bios)
- **DCS-BIOS Arduino Library:** [github.com/DCS-Skunkworks/dcs-bios-arduino-library](https://github.com/DCS-Skunkworks/dcs-bios-arduino-library)

### Protocol Documentation

- **RS-485 Protocol Wiki:** [github.com/Gadroc/dcs-bios-arduino/wiki/RS-485-Protocol](https://github.com/Gadroc/dcs-bios-arduino/wiki/RS-485-Protocol)
- **DCS-BIOS Developer Guide:** [github.com/DCS-Skunkworks/dcs-bios/.../developerguide.adoc](https://github.com/DCS-Skunkworks/dcs-bios/blob/main/Scripts/DCS-BIOS/doc/developerguide.adoc)

### Community Resources

- **OpenHornet Discord:** [discord.gg/G5PA5ju](https://discord.gg/G5PA5ju)
- **OpenHornet Website:** [openhornet.com](https://openhornet.com)
- **DCS Forums - Home Cockpits:** [forum.dcs.world](https://forum.dcs.world/forum/153-home-cockpits/)

### Hardware Vendors

- **OpenCockpit Shop (Authorized):** [opencockpitshop.com](https://opencockpitshop.com)
- **DCSBIOSKit:** [dcsbioskit.com](https://dcsbioskit.com)

### Related Projects

- **ESP32 Multi-Module DCS-BIOS:** [github.com/pavidovich/ESP32_MultiModuleDCSBios](https://github.com/pavidovich/ESP32_MultiModuleDCSBios)
- **Warthog Project:** [thewarthogproject.com](https://thewarthogproject.com/arduinos-and-dcs-bios)

---

## Appendix A: DCS-BIOS Control Reference (F/A-18C Excerpt)

### Master Arm Panel (1A2)

| Control | Type | Address | Mask | Shift | Values |
|---------|------|---------|------|-------|--------|
| MASTER_ARM_SW | Switch2Pos | 0x74C4 | 0x1000 | 12 | 0=SAFE, 1=ARM |
| MASTER_MODE_AA | Switch2Pos | 0x74C4 | 0x0800 | 11 | 0=OFF, 1=AA |
| MASTER_MODE_AG | Switch2Pos | 0x74C4 | 0x0400 | 10 | 0=OFF, 1=AG |
| EMER_JETT_BTN | Switch2Pos | 0x74C4 | 0x2000 | 13 | 0=OFF, 1=PUSH |
| FIRE_EXT_BTN | Switch2Pos | 0x74C4 | 0x4000 | 14 | 0=OFF, 1=PUSH |

### Sensor Panel (5A7A1)

| Control | Type | Address | Mask | Shift | Values |
|---------|------|---------|------|-------|--------|
| FLIR_SW | Switch3Pos | 0x74C8 | 0x3000 | 12 | 0=OFF, 1=ON, 2=STBY |
| RADAR_SW | SwitchMultiPos | 0x74C8 | 0x0E00 | 9 | 0=OFF, 1=STBY, 2=OPR, 3=EMERG |
| INS_SW | SwitchMultiPos | 0x74C8 | 0x01C0 | 6 | 0=OFF, 1=CV, 2=GND, 3=NAV, 4=IFA, 5=GYRO, 6=GB, 7=TEST |
| LTD_R_SW | Switch2Pos | 0x74C8 | 0x4000 | 14 | 0=OFF, 1=ARM |
| LST_NFLR_SW | Switch2Pos | 0x74C8 | 0x8000 | 15 | 0=OFF, 1=ON |

---

## Appendix B: Sample ESP32 Master Implementation Skeleton

```c
/**
 * ESP32 CockpitOS - ABSIS RS-485 Master
 * Skeleton implementation for DCS-BIOS compatibility
 */

#include <WiFi.h>
#include <WiFiUdp.h>
#include <HardwareSerial.h>

// Configuration
#define WIFI_SSID "your-network"
#define WIFI_PASSWORD "your-password"
#define DCS_BIOS_MULTICAST_IP "239.255.50.10"
#define DCS_BIOS_MULTICAST_PORT 5010
#define DCS_BIOS_SEND_PORT 7778

#define RS485_BAUD 250000
#define RS485_TX_PIN 17
#define RS485_RX_PIN 16
#define RS485_TXENABLE_PIN 4

#define POLL_TIMEOUT_US 2000
#define SYNC_SILENCE_US 500

// Globals
HardwareSerial RS485(1);
WiFiUDP udp;
IPAddress dcsBiosIP(127, 0, 0, 1);

uint8_t exportBuffer[1024];
size_t exportLength = 0;
bool slavePresent[128] = {false};

// Frame sync detection
enum SyncState { SYNC_WAIT, SYNC_1, SYNC_2, SYNC_3, SYNC_DONE };
SyncState syncState = SYNC_WAIT;

void setup() {
    Serial.begin(115200);

    // Initialize RS-485
    RS485.begin(RS485_BAUD, SERIAL_8N1, RS485_RX_PIN, RS485_TX_PIN);
    pinMode(RS485_TXENABLE_PIN, OUTPUT);
    digitalWrite(RS485_TXENABLE_PIN, LOW);

    // Connect WiFi
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
    }

    // Join multicast for DCS-BIOS export
    udp.beginMulticast(IPAddress(239, 255, 50, 10), DCS_BIOS_MULTICAST_PORT);
}

void loop() {
    // Task 1: Receive DCS-BIOS export data
    receiveDcsBiosExport();

    // Task 2: Poll RS-485 slaves
    pollNextSlave();
}

void receiveDcsBiosExport() {
    int packetSize = udp.parsePacket();
    if (packetSize > 0) {
        uint8_t buffer[1500];
        int len = udp.read(buffer, sizeof(buffer));
        processExportData(buffer, len);
    }
}

void processExportData(uint8_t* data, size_t length) {
    for (size_t i = 0; i < length; i++) {
        uint8_t byte = data[i];

        // Detect sync sequence: 0x55 0x55 0x55 0x55
        switch (syncState) {
            case SYNC_WAIT:
                if (byte == 0x55) syncState = SYNC_1;
                break;
            case SYNC_1:
                syncState = (byte == 0x55) ? SYNC_2 : SYNC_WAIT;
                break;
            case SYNC_2:
                syncState = (byte == 0x55) ? SYNC_3 : SYNC_WAIT;
                break;
            case SYNC_3:
                if (byte == 0x55) {
                    syncState = SYNC_DONE;
                    exportLength = 0;  // Start new frame
                } else {
                    syncState = SYNC_WAIT;
                }
                break;
            case SYNC_DONE:
                // Store export data
                if (exportLength < sizeof(exportBuffer)) {
                    exportBuffer[exportLength++] = byte;
                }
                break;
        }
    }
}

void pollNextSlave() {
    static uint8_t currentAddress = 1;
    static unsigned long lastPollTime = 0;

    // Rate limit polling
    if (micros() - lastPollTime < 1000) return;
    lastPollTime = micros();

    // Build poll packet
    uint8_t packet[256];
    size_t packetLen = 0;

    packet[packetLen++] = (0x00 << 5) | (currentAddress & 0x1F);  // Type=0, Address
    packet[packetLen++] = (uint8_t)exportLength;
    memcpy(&packet[packetLen], exportBuffer, exportLength);
    packetLen += exportLength;
    packet[packetLen++] = calculateChecksum(packet, packetLen);

    // Transmit
    setTxEnable(true);
    RS485.write(packet, packetLen);
    RS485.flush();
    setTxEnable(false);

    // Wait for response
    unsigned long start = micros();
    while (micros() - start < POLL_TIMEOUT_US) {
        if (RS485.available()) {
            handleSlaveResponse();
            slavePresent[currentAddress] = true;
            break;
        }
    }

    // Advance to next address
    currentAddress = (currentAddress % 127) + 1;
}

void handleSlaveResponse() {
    uint8_t response[256];
    size_t responseLen = 0;

    while (RS485.available() && responseLen < sizeof(response)) {
        response[responseLen++] = RS485.read();
    }

    // Parse response and forward input commands to DCS-BIOS
    if (responseLen > 2) {
        uint8_t dataLen = response[1];
        if (responseLen >= 2 + dataLen) {
            // Extract command string and send to DCS-BIOS
            sendToDcsBios(&response[2], dataLen);
        }
    }
}

void sendToDcsBios(uint8_t* data, size_t length) {
    // Send input command to DCS-BIOS via UDP
    udp.beginPacket(dcsBiosIP, DCS_BIOS_SEND_PORT);
    udp.write(data, length);
    udp.endPacket();
}

void setTxEnable(bool enable) {
    digitalWrite(RS485_TXENABLE_PIN, enable ? HIGH : LOW);
    if (!enable) {
        delayMicroseconds(10);
    }
}

uint8_t calculateChecksum(uint8_t* data, size_t length) {
    uint8_t checksum = 0;
    for (size_t i = 0; i < length; i++) {
        checksum ^= data[i];
    }
    return checksum;
}
```

---

**Document End**

*This document was compiled from analysis of the OpenHornet-Software repository, DCS-BIOS documentation, and community resources. For the latest information, consult the official repositories and Discord community.*
