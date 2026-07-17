# Hardware Design

## 1. Controller

Target controller: **ESP32-S2 Mini V1.0.0**.

The board provides:

- 3.3 V GPIO logic;
- sufficient GPIOs for four pulse inputs;
- I2C connectivity to a local controller;
- Wi-Fi connectivity to Zmartify Edge;
- USB support for development and recovery.

The final GPIO assignment shall be recorded in firmware configuration and in the wiring diagram. Avoid boot-strapping, USB and otherwise reserved pins.

## 2. Channel count

The electrical and connector design shall support four identical channels from the first revision, even when only channels 1 and 2 are initially fitted.

Suggested logical names:

```text
FLOW_CH1
FLOW_CH2
FLOW_CH3
FLOW_CH4
```

Unused inputs must have a defined idle state and must not float.

## 3. Flow-meter connector

Recommended minimum connector per channel:

| Pin | Signal | Description |
|---:|---|---|
| 1 | `+5V` | Flow-meter supply |
| 2 | `GND` | Common ground |
| 3 | `PULSE` | Pulse output |
| 4 | `RESERVED` | Future sensor signal or shield |

Connector pin order must be clearly marked. Reversing supply polarity may destroy a sensor.

## 4. Signal types

Before selecting the input circuit, identify the flow meter's output type.

### Open-collector/open-drain output

Preferred circuit:

```text
3.3 V
  |
  +-- 4.7 kOhm to 10 kOhm pull-up
  |
  +---------------- ESP32-S2 GPIO
  |
flow-meter transistor output
  |
 GND
```

The sensor may be powered from 5 V while the signal is pulled up to 3.3 V. A voltage translator is normally unnecessary in this case.

### Active 5 V push-pull output

Do not connect this signal directly to an ESP32-S2 GPIO. Use a suitable one-way 5 V to 3.3 V input stage, for example:

- a 5 V-tolerant logic buffer powered for 3.3 V output;
- a correctly dimensioned resistor divider for low-frequency clean signals;
- a transistor input stage;
- an optocoupler when galvanic isolation is desired.

A generic automatic bidirectional translator is not automatically suitable. Verify its behavior with push-pull signals, cable capacitance and the expected pulse width.

## 5. Existing voltage translator

The existing translator may be used if all of the following are true:

1. It supports the flow meter's output type.
2. Its high-voltage side accepts 5 V.
3. Its low-voltage side outputs valid 3.3 V logic.
4. It does not require an unsupported direction-control scheme.
5. It preserves the shortest expected pulse at the maximum expected frequency.
6. It remains stable with the installed cable length.

Record the translator part number in this document before finalizing the schematic.

## 6. Recommended input protection

Provide footprints for each channel:

```text
PULSE connector
    |
  1 kOhm series resistor
    |
level adaptation / pull-up
    |
ESP32-S2 GPIO
```

Optional components:

- small RC filter capacitor, initially not fitted;
- ESD protection for long or externally routed cables;
- Schmitt-trigger buffer for noisy or slow edges;
- test point at the raw and translated signal.

A capacitor can suppress noise but can also remove legitimate short pulses. Select it only after measuring the real signal.

## 7. Grounding and power

All non-isolated devices must share a common ground:

```text
5 V supply GND
ESP32-S2 GND
local controller GND
flow-meter GND 1..4
```

Use a stable 5 V supply sized for the ESP32-S2, all installed sensors and margin. Do not assume the ESP32 board's 3.3 V regulator can supply external 5 V flow meters.

Recommended practices:

- local decoupling near the ESP32-S2;
- bulk capacitance at the 5 V input;
- fused or current-limited sensor supply where appropriate;
- separate routing of pulse lines from pumps, valves and mains wiring.

## 8. I2C connection

When both controllers use 3.3 V I2C:

```text
ESP32-S2 SDA ---- local controller SDA
ESP32-S2 SCL ---- local controller SCL
ESP32-S2 GND ---- local controller GND
```

Use only one effective set of pull-ups on each bus line. Typical starting value is 4.7 kOhm to 3.3 V, subject to bus length and capacitance.

If the local controller uses a different I2C voltage, use a bidirectional open-drain-compatible level shifter.

## 9. Installation considerations

For house-water monitoring:

- use connectors appropriate for the environment;
- provide strain relief;
- keep electronics away from condensation and leaks;
- consider conformal coating or an enclosure with suitable ingress protection;
- make cable shields or reserved conductors available for future noise mitigation;
- ensure the measurement system does not compromise any certified water installation.

## 10. Hardware acceptance tests

For every channel:

1. Verify idle voltage at the ESP32 input.
2. Verify high and low levels under sensor load.
3. Measure pulse width at minimum and maximum flow.
4. Test with the intended cable length.
5. Verify no false counts during pump or valve switching.
6. Verify the unconnected channel remains stable.
7. Compare counted pulses with a signal generator or oscilloscope count.
