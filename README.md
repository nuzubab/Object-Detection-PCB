# Nano RP2040 + VL53L1X Distance Sensing System

An embedded hardware and firmware project built around the **Arduino Nano RP2040 Connect** and **VL53L1X Time-of-Flight (ToF) distance sensor**.

The system is divided into two custom PCBs:

* **Main Control Board** — Nano RP2040, power regulation, signal processing, and switched output stage
* **Sensor Board** — VL53L1X ToF sensor, I²C interface, XSHUT control, interrupt signal, and supporting circuitry

The project was designed in **KiCad** and is intended to provide programmable distance sensing with a switched output signal of approximately **11 V**.

---

## Features

* Arduino Nano RP2040 Connect based controller
* VL53L1X Time-of-Flight distance sensing
* I²C sensor communication
* Programmable distance measurement
* Short, Medium, and Long ranging modes
* Hardware-controlled VL53L1X shutdown/reset
* Optional GPIO interrupt support
* MOSFET-based 11 V output switching
* Serial Monitor debugging
* Dedicated main and sensor PCBs
* KiCad schematic and PCB design
* Modular connection between controller and sensor boards
* Firmware safety behavior when sensor initialization fails

---

## System Architecture

```text
              +----------------------+
              |      11 V Input      |
              +----------+-----------+
                         |
                         |
                 +-------v-------+
                 | Power Section |
                 | 11 V -> 5 V   |
                 +-------+-------+
                         |
                         v
             +-----------------------+
             | Arduino Nano RP2040   |
             |                       |
             | A4  <---- SDA         |
             | A5  ----> SCL         |
             | D4  ----> XSHUT       |
             | D2  <---- GPIO1       |
             | D9  ----> Output CTRL |
             +----------+------------+
                        |
              +---------v----------+
              | MOSFET Output Stage|
              | BSS138 + BSS84     |
              +---------+----------+
                        |
                        v
                    VO10 ~11 V


        MAIN BOARD                 SENSOR BOARD
   +-----------------+         +------------------+
   | Nano RP2040     |         | VL53L1X         |
   |                 |         |                  |
   | D4 -------------+-------->| XSHUT            |
   | D2 <------------+---------| GPIO1 / INT      |
   | A4 <------------+-------->| SDA              |
   | A5 -------------+-------->| SCL              |
   +-----------------+         +------------------+
```

---

## Hardware

### Main Control Board

The main PCB contains:

* Arduino Nano RP2040 Connect
* 11 V input power section
* 5 V regulator for controller power
* BSS138 N-channel MOSFET
* BSS84 P-channel MOSFET
* Sensor interface connector
* VO10 output connector
* Supporting resistors and power components

The Nano is powered from the regulated supply generated from the main input voltage.
<img width="1280" height="769" alt="2" src="https://github.com/user-attachments/assets/32a35146-b629-4fc1-b0f7-c3f87fd862bb" />

---

## Output Stage

The output is controlled by Arduino pin:

```cpp
D9
```

The simplified switching path is:

```text
Nano D9
   |
   v
BSS138
   |
   v
BSS84
   |
   v
VO10
```

### Output Logic

| D9   | Output Stage | VO10                        |
| ---- | ------------ | --------------------------- |
| LOW  | OFF          | ~0 V                        |
| HIGH | ON           | ~VCC11 / approximately 11 V |

`VO10` is intended as the external output signal.

### Output Connector

| J2 Pin | Signal |
| ------ | ------ |
| 1      | VO10   |
| 2      | GND    |

> **Important:** VO10 can reach approximately 11 V. It must never be connected directly to a 3.3 V Nano RP2040 GPIO input.

---

## Sensor Board

The sensor PCB is based on the **STMicroelectronics VL53L1X** Time-of-Flight ranging sensor.

The VL53L1X provides absolute distance measurements using infrared Time-of-Flight technology and communicates with the main controller through I²C.

Main signals used by the design:

<img width="1280" height="769" alt="1" src="https://github.com/user-attachments/assets/1c4fcdc7-28b9-45d1-bb2e-bdd2e6c027cd" />

```text
SDA
SCL
XSHUT
GPIO1 / INT
3.3 V
GND
```

---

## Main Board to Sensor Board Interface

| Nano / Main Board | Sensor Board | Function              |
| ----------------- | ------------ | --------------------- |
| D4                | XSHUT        | Sensor shutdown/reset |
| D2                | GPIO1        | Interrupt / status    |
| A4                | SDA          | I²C data              |
| A5                | SCL          | I²C clock             |
| 3.3 V             | 3.3 V        | Sensor supply         |
| GND               | GND          | Common ground         |

A common ground between both boards is essential for reliable I²C communication.

---

## VL53L1X Ranging Modes

The firmware is intended to support all three VL53L1X distance modes:

### Short Mode

Designed for shorter-distance measurements and improved immunity to strong ambient light.

```text
Mode: SHORT
```

### Medium Mode

Provides a compromise between maximum detection range and ambient-light tolerance.

```text
Mode: MEDIUM
```

### Long Mode

Provides the longest available ranging distance.

```text
Mode: LONG
```

The active mode can be selected through firmware depending on the application requirements.

---

## Firmware Operation

The intended firmware workflow is:

```text
START
  |
  v
Initialize Serial
  |
  v
Configure GPIO
  |
  v
Initialize I2C
  |
  v
Reset VL53L1X using XSHUT
  |
  v
Initialize VL53L1X
  |
  +---- Failed ----> D9 LOW / VO10 OFF
  |                       |
  |                       v
  |                 Retry Initialization
  |
 Success
  |
  v
Configure Ranging Mode
  |
  v
Start Distance Measurement
  |
  v
Read Distance
  |
  v
Apply Detection Logic
  |
  +---- Target Detected ----> D9 HIGH
  |                             |
  |                             v
  |                          VO10 ON
  |
  +---- No Target ----------> D9 LOW
                                |
                                v
                             VO10 OFF
```

---

## Basic Output Control

The output stage can be controlled using:

```cpp
const int OUTPUT_PIN = 9;

void setup() {
    pinMode(OUTPUT_PIN, OUTPUT);
    digitalWrite(OUTPUT_PIN, LOW);
}

void loop() {
    digitalWrite(OUTPUT_PIN, HIGH);
    delay(3000);

    digitalWrite(OUTPUT_PIN, LOW);
    delay(3000);
}
```

With the hardware operating correctly:

```text
D9 HIGH -> VO10 approximately 11 V
D9 LOW  -> VO10 approximately 0 V
```

---

## I²C Address

The default 7-bit I²C address expected from the VL53L1X is:

```text
0x29
```

An I²C scanner should therefore detect:

```text
I2C device found at address 0x29
```

before the distance-measurement firmware is expected to operate correctly.

---

## Safety / Fail-Safe Behaviour

The firmware should keep the external output disabled whenever the distance sensor cannot be initialized.

Example:

```cpp
while (!initSensor()) {
    setOutput(false);

    Serial.println(
        "Retrying sensor initialization in 2 seconds..."
    );

    delay(2000);
}
```

This produces:

```text
Sensor unavailable
        |
        v
     D9 LOW
        |
        v
    VO10 OFF
```

This prevents an unintended output condition when sensor communication is unavailable.

---

## Debugging

### Test 1 — Output Circuit

Test the output stage independently before testing the sensor.

```cpp
const int OUTPUT_PIN = 9;

void setup() {
    pinMode(OUTPUT_PIN, OUTPUT);
}

void loop() {
    digitalWrite(OUTPUT_PIN, HIGH);
    delay(3000);

    digitalWrite(OUTPUT_PIN, LOW);
    delay(3000);
}
```

Expected:

```text
D9 HIGH -> VO10 ≈ 11 V
D9 LOW  -> VO10 ≈ 0 V
```

If this works, the Nano-to-output switching stage is functioning.

---

### Test 2 — I²C Bus

Before running the complete firmware, scan the I²C bus.

The important expected device address is:

```text
0x29
```

for the VL53L1X.

If other I²C devices appear but `0x29` does not, the Nano I²C interface may be operational while the VL53L1X itself is not responding.

---

### Test 3 — XSHUT

Check the VL53L1X `XSHUT` signal.

```text
D4 LOW  -> Sensor shutdown
D4 HIGH -> Sensor enabled
```

After initialization, XSHUT should remain HIGH for normal communication.

---

### Test 4 — I²C Lines

Check the following signals:

```text
A4 -> SDA
A5 -> SCL
```

Typical troubleshooting checks include:

* SDA continuity
* SCL continuity
* Common GND
* Sensor supply
* XSHUT state
* I²C pull-up resistors
* Solder bridges
* VL53L1X orientation
* Connector pin order

---

## Current Development Status

The main controller and output architecture have been established.

The current debugging stage is focused on reliable VL53L1X communication.

The expected VL53L1X address is:

```text
0x29
```

During previous testing, the I²C scanner detected:

```text
0x60
0x6A
```

while:

```text
0x29
```

was not detected.

Therefore, the current troubleshooting focus is:

```text
Sensor power
     +
Common ground
     +
XSHUT
     +
SDA / SCL
     +
Pull-ups
     +
Sensor soldering
     |
     v
VL53L1X detection at 0x29
```

Once `0x29` is detected reliably, the complete distance-detection and output-control firmware can be validated.

---

## Development Tools

### Hardware Design

* KiCad
* Custom PCB design
* Digital multimeter
* Oscilloscope / logic analyzer for advanced debugging

### Firmware

* Arduino IDE
* Arduino Nano RP2040 Connect
* C / C++
* Wire / I²C communication
* VL53L1X driver

---

## Recommended Test Sequence

For easier debugging, test each subsystem independently:

```text
1. Verify 11 V input
        ↓
2. Verify regulated controller supply
        ↓
3. Verify Nano RP2040 operation
        ↓
4. Test D9 output
        ↓
5. Verify VO10 switching
        ↓
6. Verify sensor 3.3 V supply
        ↓
7. Verify XSHUT
        ↓
8. Scan I²C
        ↓
9. Confirm VL53L1X at 0x29
        ↓
10. Read distance values
        ↓
11. Test Short / Medium / Long modes
        ↓
12. Integrate distance detection with VO10
```

---

## Project Goals

The main goals of this project are to:

1. Develop a custom embedded distance-sensing platform.
2. Interface the VL53L1X directly with an Arduino Nano RP2040.
3. Support multiple programmable ranging modes.
4. Generate an external switched signal based on sensor measurements.
5. Separate the sensor and controller into modular PCBs.
6. Provide reliable firmware-level fault handling.
7. Develop a platform suitable for further embedded sensing experiments.

---

## Future Improvements

Planned improvements may include:

* Configurable detection threshold
* Runtime Short / Medium / Long mode switching
* Measurement filtering
* Hysteresis around the detection threshold
* Adjustable ON/OFF delay
* Sensor timeout handling
* Interrupt-based measurement
* Improved startup diagnostics
* Calibration support
* More detailed Serial Monitor diagnostics
* Error/status codes
* Hardware revision documentation
* PCB manufacturing files
* Automated firmware tests

---

## References

Technical implementation is based on documentation for:

* **Arduino Nano RP2040 Connect** — Arduino
* **VL53L1X Time-of-Flight Ranging Sensor** — STMicroelectronics
* **VL53L1X Datasheet / API Documentation** — STMicroelectronics
* **KiCad Schematic and PCB Design Documentation** — KiCad

---

## Disclaimer

This repository represents an embedded hardware prototype under active development.

Always verify:

* Supply voltages
* GPIO voltage levels
* MOSFET pinouts
* Connector orientation
* PCB continuity
* Sensor supply requirements

before connecting external hardware.

In particular, the `VO10` output can be significantly higher than the Nano RP2040's GPIO voltage and must not be connected directly to a 3.3 V GPIO.

---

## Contributions

Suggestions, bug reports, hardware improvements, and firmware improvements are welcome.

If you find an issue, please open a GitHub Issue with:

* Hardware revision
* Firmware version
* Serial Monitor output
* Measured voltages
* Description of the problem
