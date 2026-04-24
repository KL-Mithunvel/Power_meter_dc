# ESP32 Inline Power Meter Module — Objective, Specs, Requirements (v2)

## 1. Objective

Build a reusable **inline DC power meter module** that can be inserted between a DC power source (battery / adapter) and a project load to measure and report:

* **Voltage (V)** of the main power rail
* **Current (A)** drawn by the load (via external shunt)
* **Power (W = V×I)** consumed by the load

The module must:

* Power only its **ESP32 + OLED** from an onboard **buck converter** (**no buck power routed to the load output**).
* Provide a robust **pass-through high-current power path** from input connectors to an output screw terminal.
* Offer selectable communications output from the ESP32 via a **mode button**, with the current mode shown on the OLED:

  * **UART** (wired)
  * **I2C** (wired)
  * **Bluetooth** (wireless)
* Provide **galvanic isolation** on all **wired** communications (UART + external I2C) to protect host devices.

## 2. Non-Goals

* Not a charger, BMS, or power supply for the load.
* Not intended to power an external host device.
* Not intended to use USB-UART as the external interface (UART means logic-level UART signals).

## 3. System Overview

### 3.1 Main Power Flow (High-Current Path)

**High-current input (XT60 / IN screw)** → **(optional high-current fuse)** → **external shunt** → **OUT screw terminal** → load

Notes:

* The external shunt is in the main path. The PCB reads the shunt using Kelvin sense leads.
* The PCB should not be the primary 80 A conductor (use wiring/busbar strategy).

### 3.2 Module Power (Electronics Path)

Main input rail (VIN) → protection → **buck converter (5 V)** → ESP32 + OLED + logic

Notes:

* Buck output powers only ESP32/OLED.
* Electronics branch includes its own small fuse/polyfuse independent of the 80 A fuse.

## 4. Electrical Interfaces

### 4.1 Input Connectors

**High-current inputs (intended for up to 80 A):**

* XT60 (or higher-current alternative depending on final connector decision)
* Screw terminal input (rating TBD; must be validated for 80 A)

**Low-current convenience input:**

* DC barrel jack (**NOT for 80 A path**)

**Requirement:** prevent accidental use of barrel jack in an 80 A setup.
Recommended implementations:

* Mechanical/PCB policy: **barrel jack powers electronics only** (not tied to the 80 A pass-through rail), **or**
* Use a **selector/jumper/solder-jumper** so the user explicitly chooses:

  * LOW-CURRENT IN (barrel), or
  * HIGH-CURRENT IN (XT60/screw)

### 4.2 Output Connector

* 2-pin screw terminal output: OUT+/OUT− (high-current path; rating TBD)

### 4.3 Communications Connector (Isolated Domain)

Wired comms must be **galvanically isolated**. External host connects only to the **isolated-side** pins.

* **UART (isolated):** TX_ISO, RX_ISO, GND_ISO
* **External I2C (isolated):** SDA_EXT_ISO, SCL_EXT_ISO, GND_ISO
* Optional: **VISO (3.3 V)** pin (see Section 6.5 power for isolated side)

Bluetooth is wireless and requires no connector.

### 4.4 Mode Select

* One tactile button cycles modes: **UART → I2C → Bluetooth → repeat**
* Mode is shown on OLED
* Mode persists in ESP32 NVS (recommended)

### 4.5 Dual I2C Bus Architecture (REQUIRED)

To keep internal sensing/display reliable while providing I2C to a host, the ESP32 uses two independent I2C buses:

1. **Internal I2C (ESP32 = master, non-isolated)**

* Devices: shunt monitor IC + OLED
* Lines: SDA_INT / SCL_INT
* On-board pull-ups populated (typ. 4.7k to 3.3 V)

2. **External I2C (ESP32 = slave, isolated)**

* Host communication only
* Lines on header: SDA_EXT_ISO / SCL_EXT_ISO
* Pull-ups handled on the isolated side (configurable) and must not fight host pull-ups

## 5. Measurements

### 5.1 Voltage Measurement

* Prefer reading bus voltage from the shunt monitor IC (best accuracy, less ADC drift).
* Alternative: voltage divider into ESP32 ADC.

### 5.2 Current Measurement (LOCKED)

Inline current via **external bolt-down shunt**.

Implementation requirements:

* Use a **75 mV @ 100 A** class shunt (headroom for 80 A operation).
* Use **Kelvin (4-wire) sense** from shunt to PCB.
* A shunt monitor IC measures shunt voltage and bus voltage; ESP32 computes current and power.

Shunt monitor IC options:

* **INA228 (preferred)**
* **INA226 (acceptable for 24 V systems; 36 V bus max)**

Outputs:

* Voltage (V)
* Current (A)
* Power (W)
* Optional: energy integration (Wh, mAh)

### 5.3 Display

OLED shows at minimum:

* Voltage (V)
* Current (A)
* Power (W)
* Mode (UART / I2C / BT)

Optional:

* Energy (Wh), mAh
* Peak current
* Min/max voltage

## 6. Safety Requirements

### 6.1 High-Current Path Safety

* Main path components must be rated for **12–24 V, 80 A** continuous (connector, wire/busbar, shunt mounting).
* Keep high-current routing physically separated from low-voltage logic.

### 6.2 Input Protection (Electronics)

At minimum:

* Reverse polarity protection (MOSFET ideal diode preferred)
* TVS diode for transients (selected for 24 V systems with margin)
* Bulk + decoupling capacitors near inputs

### 6.3 Grounding / Noise Control

* Shunt Kelvin sense routing must be short, symmetric, and away from switching nodes.
* Keep buck converter switching loops tight.

### 6.4 Host Protection (Electrical)

* Wired interfaces are isolated; do not require tying host ground to module ground.
* Add ESD protection near external headers (recommended).

### 6.5 Galvanic Isolation (REQUIRED for wired comms)

Goal: protect hosts (e.g., Raspberry Pi) from ground shifts, fault currents, and transients from the 80 A power domain.

Requirements:

* **UART and external I2C are galvanically isolated**.
* External header uses isolated reference **GND_ISO**.
* The isolation barrier includes both signal isolation and isolated-side powering.

Implementation guidance:

* UART: digital isolator IC(s) with adequate data rate for chosen baud.
* I2C: use an I2C-capable isolator (bidirectional) or isolator + proper buffering.
* Power for isolated side:

  * Preferred: onboard **isolated DC-DC** to generate **VISO (3.3 V)**.
  * Alternative: host supplies **VISO_IN (3.3 V)** (only if explicitly accepted by team; isolation is conditional).

Notes:

* Bluetooth mode is inherently isolated (wireless).

## 7. Target Specifications (Locked/Chosen)

### 7.1 Electrical

* VIN range: **12–24 V DC**
* Main path current (continuous target): **80 A**
* Electronics supply: **buck to 5 V** for ESP32/OLED only

### 7.2 Mechanical (TBD)

* Board dimensions, mounting holes, connector placement

### 7.3 Firmware / UX

* OLED refresh rate: TBD
* Mode selection: short press cycles modes; store in NVS

## 8. Data Protocol Requirements

### 8.1 UART (isolated)

* Default: 115200 8N1 (configurable)
* Output line example:

  * `V=12.34,I=1.25,P=15.43,MODE=UART`

### 8.2 I2C (isolated external bus)

* ESP32 acts as **I2C slave** on SDA_EXT_ISO/SCL_EXT_ISO
* Default address suggestion: **0x42** (final TBD)
* Register map (TBD) exposes:

  * Voltage (mV)
  * Current (mA)
  * Power (mW)
  * Mode
  * Status flags

### 8.3 Bluetooth

* BLE recommended
* Characteristics:

  * Voltage (mV)
  * Current (mA)
  * Power (mW)
  * Mode

## 9. Implementation Notes / Sizing Guidance

### 9.1 Buck Converter

* Target: 5 V dedicated rail
* Output capability: **≥ 1 A** recommended
* Input rating: **≥ 36 V** recommended (margin above 24 V)
* Add separate small fuse/polyfuse on buck input: ~0.5–1.5 A class

### 9.2 Fusing

Two domains:

1. **Main high-current path**

* If included, use ANL/MIDI/MEGA fuse style
* Rating depends on wiring/connector limits; for an 80 A continuous target, typical selection is in the **80–100 A** class, validated by wire gauge and expected overload profile.

2. **Electronics branch**

* Small fuse/polyfuse ~**1 A** class to protect buck/ESP/OLED.

### 9.3 80 A Mechanical/PCB Strategy

* Do not route 80 A through standard PCB copper.
* Use wiring/busbar for the main path and mount shunt mechanically.
* PCB carries only Kelvin sense, buck feed, and logic.

## 10. Remaining Decisions

* Final shunt monitor IC: **INA228 vs INA226**
* Final connector set for true 80 A use (XT60 vs higher-current connector; screw terminal rating)
* Isolated-side power approach: onboard isolated DC-DC vs host-supplied VISO
* OLED type/size

---

## Appendix A — Quick Reference

* VIN: 12–24 V
* Main path: 80 A
* Current sensing: external 75 mV / 100 A shunt + monitor IC
* Buck: 5 V, ≥ 1 A (ESP/OLED only)
* Wired comms: UART + I2C **galvanically isolated**
* Bluetooth: BLE
