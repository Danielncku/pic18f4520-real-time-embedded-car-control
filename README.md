
## Project Overview
This project implements an embedded **autonomous car control system** using the **PIC18F4520 microcontroller**.  
The system integrates multiple modules including **HC-05, DC motors, ultrasonic sensor, servo motor, buzzer, laser module, and TM1637 7-segment display**.

The car can be controlled via **UART commands** and includes safety mechanisms such as obstacle detection and automatic stop.

---

## Features
- UART remote control (Forward / Backward / Left / Right / Stop)
- Ultrasonic obstacle detection
- Distance-based buzzer warning
- Servo scanning with laser activation
- 4-digit 7-segment distance display
- Automatic motor stop when obstacle is too close

---

## Hardware
- PIC18F4520
- HC-05 Bluetooth Module
- HC-SR04 Ultrasonic Sensor
- DC Motors + Motor Driver
- Servo Motor
- Buzzer
- Laser Module
- TM1637 4-digit 7-segment display
- UART / Bluetooth module

---

## How to Run

### 1. Build
Compile the project using **MPLAB X IDE** with **XC8 compiler**.

### 2. Upload
Flash the firmware to the **PIC18F4520** using a PIC programmer (PICkit).

### 3. Connect Serial
Open a serial terminal:
