# PIC18F4520 Real-Time Embedded Car Control

> Final project for Computer Science embedded systems course.
> This project combines Bluetooth control, obstacle detection, warning feedback, and multi-module integration on a PIC18F4520.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Main Features](#main-features)
- [Hardware Setup](#hardware-setup)
- [Software Architecture](#software-architecture)
- [Control Flow](#control-flow)
- [Module Integration Notes](#module-integration-notes)
- [Problems We Faced and How We Solved Them](#problems-we-faced-and-how-we-solved-them)
- [Build and Run](#build-and-run)
- [Command Reference](#command-reference)
- [What We Learned](#what-we-learned)
- [Future Improvements](#future-improvements)

---

## Project Overview

This project is a real-time embedded car control system built on the **PIC18F4520** microcontroller. Our original goal was to make a basic Bluetooth-controlled car, but during development we realized that only receiving commands was not enough. If the car could move but had no sensing or protection logic, it was too easy for it to crash or behave unpredictably.

So we extended the design into a more complete embedded system. The final version includes:

- UART/Bluetooth remote control through **HC-05**
- obstacle detection using **HC-SR04**
- warning feedback using a **buzzer**
- distance display on a **TM1637 4-digit 7-segment display**
- servo-driven motion with a **laser interaction sequence**
- basic safety logic to stop or back off when the front distance becomes dangerous

What made this project interesting was not just getting each part to work separately. The real challenge was making all of them run on the same MCU with reasonable timing behavior.

---

## Main Features

### Real-time control

- Accepts single-character UART commands from Bluetooth
- Supports forward, backward, left, right, and stop control
- Stops automatically if movement commands are not refreshed in time

### Sensor-based protection

- Continuously checks front distance using the ultrasonic sensor
- Blocks forward movement when an obstacle is too close
- Triggers a safer response if danger lasts for a period of time

### User feedback

- Uses a buzzer to indicate distance-based warning levels
- Shows measured distance on the TM1637 display
- Prints debug messages through UART for system status

### Extra interaction module

- Activates a laser and servo sweep sequence on command
- Demonstrates PWM control and timed event handling

---

## Hardware Setup

| Module | Purpose |
| --- | --- |
| PIC18F4520 | Main controller |
| HC-05 Bluetooth | Wireless UART command input |
| HC-SR04 Ultrasonic Sensor | Front obstacle detection |
| DC Motors + Motor Driver | Car movement |
| Servo Motor | Rotational motion control |
| Laser Module | Visual interaction output |
| Buzzer | Distance warning feedback |
| TM1637 4-digit Display | Distance display |

---

## Software Architecture

Even though the code is currently implemented in a single `main.c` file, the project is logically divided into several subsystems.

### 1. UART command subsystem

- Receives Bluetooth commands through UART
- Uses a **high-priority interrupt** for fast response
- Converts incoming characters into motor or action commands

### 2. Motor control subsystem

- Controls direction pins for forward, backward, left, right, and stop
- Tracks movement state using software variables
- Adds timeout-based stopping to prevent unintended continuous motion

### 3. Ultrasonic sensing subsystem

- Sends trigger pulses to HC-SR04
- Measures echo pulse width
- Converts pulse duration into distance
- Updates system safety logic periodically

### 4. Buzzer subsystem

- Uses **Timer1 low-priority interrupt** to generate buzzer toggling
- Uses a small software state machine to control beep patterns
- Changes warning frequency based on obstacle distance

### 5. Display subsystem

- Uses bit-banging to control the TM1637 display
- Updates distance only when the value changes
- Briefly protects write timing during display transmission

### 6. Servo and laser subsystem

- Uses PWM to control the servo angle
- Runs a short laser-assisted sweep sequence when triggered
- Automatically turns the laser off after the active window

---

## Control Flow

The system uses a mixed real-time design instead of doing everything in one blocking loop.

### Interrupt-driven tasks

- **High-priority interrupt**
  UART receive

- **Low-priority interrupt**
  Timer1 buzzer waveform generation

### Main loop tasks

- command handling
- movement timeout checking
- laser timeout checking
- ultrasonic distance update
- buzzer policy update
- 7-segment display update
- software time base update

### Simplified runtime logic

```text
Bluetooth command received
-> UART interrupt stores command
-> main loop reads command
-> motor state is updated

Every 100 ms
-> read ultrasonic distance
-> decide stop / warning / safe behavior
-> update buzzer state
-> update display

Every 10 ms
-> increment software system time
-> check movement timeout
-> check laser timeout
```

This design is simple, but for a course project it was a practical way to coordinate multiple peripherals without needing a full RTOS.

---

## Module Integration Notes

This part was honestly the hardest part of the whole project.

Getting one module to work in isolation was usually not too bad. The trouble started after we connected everything together. A feature that looked stable by itself could suddenly become unreliable once another module with different timing requirements was added.

### Why integration was difficult

- UART wanted immediate response
- ultrasonic reading could block if not handled carefully
- buzzer output needed repeated timing events
- TM1637 communication needed stable bit-level timing
- servo movement introduced delay while sweeping
- safety logic had to override some manual commands

### How we handled it

- We gave **UART** the highest priority because command latency matters during movement.
- We separated **buzzer waveform generation** from **warning decision logic**.
- We used a **software time base** instead of putting large delays everywhere.
- We let the **main loop** handle periodic decisions, while interrupts handled tasks that needed faster timing.
- We used **shared state variables** to connect modules instead of tightly coupling every function.

### Practical result

This structure was not perfect, but it made the system much easier to debug. Once we stopped thinking of the project as "one giant loop" and started thinking in terms of separate timed responsibilities, the behavior became much more predictable.

---

## Problems We Faced and How We Solved Them

### 1. The car could keep moving when command updates stopped

At an early stage, if Bluetooth communication became unstable, the last movement state could remain active longer than we wanted. That was risky because the car might continue moving even when the user had already released the control button.

**What we did:**

- added a movement timeout
- required commands to be refreshed continuously
- forced the car to stop if no fresh command arrived in time

This small change improved safety a lot.

### 2. Forward command and obstacle detection could conflict

The user might send `F`, but the ultrasonic sensor might already detect an obstacle directly in front of the car. If we always trusted the command, the car would still crash. If we always trusted the sensor too aggressively, the control experience became annoying.

**What we did:**

- blocked forward motion when distance became too short
- used different distance zones instead of a single threshold
- added a longer-term danger condition for more serious protection

This gave us a more reasonable balance between manual control and autonomous safety.

### 3. The buzzer logic became messy with direct on/off control

At first, it was tempting to just turn the buzzer on whenever the object was close. But that made the behavior too rough, and timing control became difficult once other modules were active at the same time.

**What we did:**

- used Timer1 interrupt to generate the buzzer toggle signal
- used a software state machine for `BUZZ_IDLE`, `BUZZ_ON`, and `BUZZ_OFF`
- changed the beep interval depending on distance

That separation made the warning behavior easier to control and easier to tune.

### 4. TM1637 updates were more sensitive than we expected

The TM1637 display looks simple, but since it is bit-banged, timing quality matters. When interrupt activity and display writes overlapped in the wrong moment, the output could become unstable.

**What we did:**

- updated the display only when the shown value changed
- kept display writes short
- protected timing during the actual write section

This reduced display glitches and unnecessary communication overhead.

### 5. The ultrasonic sensor was not always as clean as in lab examples

In real testing, sensor data changed depending on surface angle, object material, and movement of the car itself. The sensor could also block the flow if echo timing was not handled carefully.

**What we did:**

- sampled the sensor periodically instead of constantly
- added a safety break during echo measurement
- treated the sensor as a short-range protection tool, not as a perfect measurement device

That was a pretty important mindset shift for us.

### 6. Smooth servo movement introduced extra blocking time

We implemented gradual servo movement because it looked better than jumping directly to the target. But in exchange, it also consumed time and could delay other actions.

**What we learned from this:**

- smooth output is nice
- but non-blocking design matters more when the system grows

If we continue this project, servo behavior is one of the first parts we would redesign.

---

## Build and Run

### Development environment

- **MPLAB X IDE**
- **XC8 compiler**
- **PICkit** or another compatible PIC programmer

### Build steps

```text
1. Open the source/project in MPLAB X IDE
2. Select PIC18F4520 as the target device
3. Build the code with XC8
4. Flash the firmware to the board
5. Connect the HC-05 module
6. Pair Bluetooth with a serial terminal app
7. Send single-character commands
```

### UART behavior

The project uses serial command input at a typical embedded control style. Commands are sent one character at a time, and the firmware reacts immediately through the main loop after UART interrupt reception.

---

## Command Reference

| Command | Description |
| --- | --- |
| `F` | Move forward |
| `B` | Move backward |
| `L` | Turn left |
| `R` | Turn right |
| `S` | Stop |
| `H` | Turn on laser and run servo sweep action |

---

## What We Learned

This project taught us a lot more than just how to control motors with a PIC.

From a Computer Science student perspective, the most valuable part was learning how different low-level topics actually interact in one system:

- interrupt priority affects responsiveness
- blocking code affects real-time behavior immediately
- hardware modules that work alone may still fail after integration
- safety logic should be planned early, not added at the very end
- embedded debugging often means checking both software logic and hardware timing

One thing we really felt during this project was that embedded systems are not difficult because each individual concept is impossible. The hard part is that everything has to coexist on limited hardware, and small timing decisions can change the behavior of the whole system.

---

## Future Improvements

- Refactor `main.c` into separate modules
- Redesign servo behavior as a non-blocking state machine
- Add distance filtering or moving average
- Improve display formatting for clearer decimal output
- Separate manual mode and safety override logic more cleanly
- Add better documentation for wiring and pin mapping

---

## Final Notes

This project ended up being more than a simple remote-control car. For us, it became a small but very practical example of embedded system integration. We had to think about timing, interrupts, module interaction, and safety behavior all at once.

Even though the code structure can still be improved, the project gave us a much better understanding of what "real-time embedded programming" actually feels like in practice.
