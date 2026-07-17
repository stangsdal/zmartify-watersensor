# Hardware Design

## 1. Controller

Target controller: **ESP32-S2 Mini V1.0.0** using 3.3 V GPIO logic.

The board shall support:

- four pulse-based flow inputs;
- one or more temperature-sensor buses;
- I2C communication with the irrigation controller;
- Wi-Fi communication with Zmartify Edge;
- USB development and recovery;
- reserved expansion for future sensor interfaces.

Final GPIO assignments must avoid boot-strapping, USB and otherwise reserved pins and must be recorded in firmware configuration and a wiring table.

## 2. Flow inputs

Provide four identical logical flow channels from the first revision, even when only two are populated.

Recommended connector:

| Pin | Signal | Description |
|---:|---|---|
| 1 | `+5V` | Flow-meter supply |
| 2 | `GND` | Common ground |
| 3 | `PULSE` | Pulse output |
| 4 | `RESERVED` | Shield or future signal |

Unused inputs must have a defined idle state.

### Open-collector/open-drain outputs

Use a pull-up to 3.3 V, typically 4.7–10 kOhm. The sensor may be powered from 5 V while its transistor output is pulled up to 3.3 V.

### Active 5 V push-pull outputs

Never connect directly to an ESP32-S2 GPIO. Use a verified one-way 5 V to 3.3 V stage, resistor divider, transistor stage, suitable buffer or optocoupler.

The existing voltage translator may be used only after confirming its part number, direction behavior, voltage limits, pulse-width performance and stability with the installed cable length.

### Protection per flow input

Provide footprints for:

- approximately 1 kOhm series resistance;
- optional RC filtering, initially not fitted;
- optional ESD protection;
- optional Schmitt-trigger conditioning;
- raw and translated test points.

## 3. Temperature sensors

The preferred initial sensor is externally powered DS18B20 using three conductors:

| Signal | Connection |
|---|---|
| `VDD` | 3.3 V |
| `GND` | common ground |
| `DATA` | ESP32-S2 1-Wire GPIO |

Use an external pull-up, initially 4.7 kOhm from `DATA` to 3.3 V. Avoid parasite power for field installations.

### Bus topology

1-Wire works best as a trunk with short branches. Large star topologies and long unterminated branches should be avoided. For sensors in physically separate areas, reserve the option of multiple independent 1-Wire buses.

For each bus provide:

- a connector with 3.3 V, GND and DATA;
- a replaceable or configurable pull-up;
- optional small series resistance near the controller;
- ESD protection when cables leave the enclosure;
- a data test point;
- local decoupling near sensors where practical.

The actual number of supported temperature sensors shall be configurable and bounded by firmware memory, timing and cable characteristics rather than hard-coded to the flow-channel count.

## 4. Future sensor expansion

Reserve GPIO, power budget and connector strategy for later sensor providers such as:

- soil-moisture probes;
- pressure transducers;
- leak contacts;
- tank-level sensors;
- conductivity sensors;
- local air temperature and humidity.

No universal electrical input is assumed. Future boards may need dedicated analog front ends, switched sensor power, external ADCs, protected digital inputs or additional buses.

For analog soil-moisture sensors, do not connect unknown 5 V analog outputs directly to the ESP32 ADC. Verify output range, source impedance, noise, corrosion behavior and calibration needs. Prefer corrosion-resistant capacitive probes over continuously energized resistive probes.

## 5. Grounding and power

All non-isolated devices must share a common ground. Use a stable 5 V supply sized for the ESP32-S2, installed sensors and margin.

Recommended practices:

- local decoupling near the ESP32-S2;
- bulk capacitance at the 5 V input;
- current-limited or fused external sensor supply where appropriate;
- separate routing from pumps, valves and mains wiring;
- optional switched power for sensors that should not be continuously energized;
- adequate enclosure and condensation protection.

## 6. I2C connection

When both controllers use 3.3 V I2C, connect SDA, SCL and GND directly and use one effective set of pull-ups. A typical starting value is 4.7 kOhm, subject to bus capacitance and length.

If voltage domains differ, use a bidirectional open-drain-compatible level shifter.

## 7. Safety boundary

Temperature values sent over I2C may support irrigation decisions, frost protection and monitoring, but a remote digital sensor must not be the only protection for safety-critical overheating, pump damage or similar hazards. Use local controller interlocks or dedicated hardware protection where required.

## 8. Acceptance tests

### Flow inputs

- verify idle and active logic levels;
- measure minimum pulse width and maximum frequency;
- test intended cable lengths;
- verify no false counts during pump and valve switching;
- compare counts with a signal generator or oscilloscope.

### Temperature buses

- enumerate and record stable sensor ROM IDs;
- verify CRC and disconnect detection;
- test minimum and maximum intended cable lengths;
- verify measurements while flow interrupts, Wi-Fi and I2C are active;
- test power cycling and sensor replacement;
- verify no bus fault blocks the remaining firmware.

### Expansion

- confirm reserved GPIO and power budget in each hardware revision;
- document voltage range and protection before connecting a new sensor type.
