# EZB V1.1 Controller Hardware

**Revision:** V1.1 (ESP32 + LicheeRV Hybrid/Cube A7Z)

**Status:** Prototype

**Maintainer:** Niko Bagović

## 1. System Architecture

The EZB V1 is a custom carrier and control board designed for the **LicheeRV Nano (MaixCam)** or **Cubie A7Z** AI module. It integrates high-power illumination, UWB localization, and motion sensing into a handheld form factor.

### Core Components
* **Application Processor:** LicheeRV Nano/Cube A7Z (Linux/MaixPy)
    * *Role:* Computer Vision, AI Inference, WiFi Streaming, Haptic Control, User Logic
* **Real-time Controller:** ESP32-WROVER-E
    * *Role:* Sensor Fusion (IMU/UWB), Power Management, LED PWM.
* **UWB Transceiver:** Qorvo DW3220
    * *Interface:* SPI (Shared Bus).
* **IMU:** TDK ICM-42605 6-Axis Gyro/Accel
    * *Interface:* SPI (Shared Bus).
* **Power:** 2S Li-Ion Input (8.4V max) with dedicated Boost/Buck regulators.

---

## 2. Pin Mapping & Interfaces

### ESP32-WROVER-E Pinout
The ESP32 acts as the SPI Master for on-board sensors and manages all real-time I/O.

| GPIO | Function | Net Label | Description |
| :--- | :--- | :--- | :--- |
| **IO13** | **MOSI** | `SPI_MOSI` | Shared SPI Data Out |
| **IO12** | **MISO** | `SPI_MISO` | Shared SPI Data In |
| **IO14** | **SCK** | `SPI_CLK` | Shared SPI Clock |
| **IO27** | **CS** | `UWB_CS` | Chip Select for DW3220 |
| **IOXX** | **CS** | `IMU_CS` | Chip Select for ICM-42605 |
| **IO2** | **PWM** | `LED_PWM` | LED Driver Dimming Signal |
| **IO25** | **ADC** | `VBAT_DET` | Battery Voltage Sense (1/3 Divider) |
| **IO26** | **GPIO** | `SWITCH_DET` | Input from User Switch |
| **IO23** | **IRQ** | `IMU_INT2` | IMU Motion Interrupt |
| **IO15** | **GPIO** | `NSS` | CS to LicheeRV |
| **IO22** | **GPIO** | `LED_PWM1` | Power Status LED Control |
| **IO17** | **GPIO** | `LED_PWM2` | Power Status LED Control |
| **IO16** | **GPIO** | `LED_PWM3` | Power Status LED Control |

### LicheeRV Nano Header
The LicheeRV sits on a dedicated header connecting it to the power rails and the ESP32 for inter-processor communication.

| Pin | Net Name | Function |
| :--- | :--- | :--- |
| **7** | `MISO` | SPI MISO (Slave to ESP32) |
| **5** | `MOSI` | SPI MOSI (Slave to ESP32) |
| **4** | `CLK` | SPI CLK (Slave to ESP32) |
| **8** | `NSS` | Chip Select  |
| **1+2** | `5V_VDD` | Main Power Input |
| **14** | `3V3_VDD` | 3v3 Output |
| **8** | `Motor Control` | Haptic Motor Gate |
| **13** | `LED_BOOST_CNV_ON` | Turning on the 12V Boost conv needed for the LED driver |
| **--** | `MIPI_CSI` | Camera Connector (Top side FPC) |
---
### Cube A7Z
The Cube A7Z sits on a dedicated header connecting it to the power rails and the ESP32 for inter-processor communication.

| Pin | Net Name | Function |
| :--- | :--- | :--- |
| **21** | `MISO` | SPI MISO (Slave to ESP32) |
| **19** | `MOSI` | SPI MOSI (Slave to ESP32) |
| **23** | `CLK` | SPI CLK (Slave to ESP32) |
| **24** | `NSS` | Chip Select  |
| **2+4** | `5V_VDD` | Main Power Input |
| **1** | `3V3_VDD` | 3v3 Output |
| **8** | `Motor Control` | Haptic Motor Gate |
| **3** | `LED_BOOST_CNV_ON` | Turning on the 12V Boost conv needed for the LED driver |
| **--** | `MIPI_CSI` | Camera Connector (Top side FPC) |
---
## 3. Power Distribution Network

The system is designed for a **2S Li-Ion Battery Pack (7.4V Nominal / 8.4V Max)**.

### Voltage Rails
1.  **VBAT (6.0V - 8.4V):** Raw battery input. Fused.
2.  **12V Boost Rail (TPS61288):** Steps VBAT up to 12V to drive the series LED string.
    * *Note:* This rail is dedicated to the illumination system.
3.  **5V System Rail (TPS54335):** Buck converter. Powers the LicheeRV Nano, USB peripherals, and charging logic.
4.  **3.3V Logic Rail (LDO):** Low-noise power for ESP32, UWB, and IMU logic.

### Charging Subsystem
* **IC:** BQ25886 Management IC.
* **Configuration:** 2-Cell (8.4V termination).
* **Input:** USB-C (5V). The BQ25886 boosts 5V input to charge the 2S HV pack together with smart charging (precharge etc..)

---

## 4. Subsystem Details

### Illumination (LED Driver)
* **Driver IC:** AL8843.
* **Topology:** Constant Current Buck/Linear.
* **Control:** * Driven by `IO2` on the ESP32.
    * **Logic:** Proportional Duty Cycle (100% = Max Brightness, 0% = Off).
* **Thermal:** LEDs are mounted on a companion Aluminum MCPCB (`LED_HeatSink_PCB`) connected via wire harness.

### Ultra-Wideband (UWB)
* **Transceiver:** Qorvo DW3220.
* **Interface:** SPI Mode 0.
* **Clocking:** On-board 38.4MHz XTAL.
* **RF Path:** Routes to either an on-board ceramic antenna or u.FL connector depending on BOM variant.

### Motion Sensing (IMU)
* **Sensor:** TDK InvenSense ICM-42605.
* **Features:** 6-Axis Gyroscope + Accelerometer.
* **Connection:** Shared SPI bus. Interrupts routed to ESP32 for wake-on-motion functionality.

## 5. Mechanical & Assembly
* **PCB:** 4-Layer FR4.
* **Camera:** MIPI CSI sensor connects via the custom `Camera_to_AI_Flex_Pcb` FPC cable to the LicheeRV Nano.
* **Haptics:** Solder pads provided on the bottom layer for a 2-wire vibration motor.
* **High Power LEDs** Connector provided on the top layer
* **Switch:** Solder pads provided on the bottom layer for a 2-wire switch to include dim modes.

## 6. Future Improvements Suggestions 

Based on testing results and future requirements for production units using fixed battery chemistries:

1.  **Full BMS Implementation (18650 Support):**
    * Re-integrate hardware (removed as we temporarily went to AA batteries) **Over-Voltage Protection (OVP)**, **Under-Voltage Protection (UVP)**, and **Cell Balancing** circuits.
    * *Purpose:* Required for safe operation when reverting to standard 2S Li-Ion 18650 battery packs (replacing the current variable input support).

2.  **State-of-Charge (SoC) Fuel Gauge:**
    * Replace the simple voltage divider (`VBAT_DET`) with a dedicated I2C Fuel Gauge IC (e.g., Maxim or TI series).
    * *Purpose:* Provide accurate 0-100% battery reporting to the user interface, independent of voltage sag under LED load.

3.  **Connectorization:**
    * Replace bottom-side solder pads (Switch, Haptics, Battery) with vertical JST/Molex headers.
    * *Purpose:* Simplify final assembly and allow for modular replacement of mechanical components without soldering.

4.  **USB-PD Fast Charging:**
    * Upgrade the TP5100 linear charging logic to a USB-PD compliant switching charger.
    * *Purpose:* Reduce charge time for high-capacity 18650 cells.
      
<img width="1035" height="711" alt="Screenshot_1" src="https://github.com/user-attachments/assets/146eb758-a32b-443c-aaa3-31238fb26421" />
<img width="1039" height="712" alt="Screenshot_2" src="https://github.com/user-attachments/assets/9592af7a-26b1-4b8b-a2d7-4d55b715cfdd" />
