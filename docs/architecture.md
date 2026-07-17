# System Architecture

## 1. Purpose

`zmartify-watersensor` is a reusable ESP32-S2 Mini sensor node for water and irrigation installations.

The node shall initially:

- count pulses from up to four flow meters;
- calculate current flow and accumulated volume;
- acquire temperature measurements;
- preserve flow totals and configuration across restarts;
- expose local measurements over I2C;
- publish measurements over Wi-Fi to Zmartify Edge;
- remain operational when Wi-Fi, MQTT, the Edge server or the I2C master is unavailable.

The architecture shall later support soil moisture, pressure, leak, level, conductivity and related sensors without replacing the core measurement or communication layers.

## 2. System context

```text
Flow meters 1..4 ---- pulse input adapters ----+
                                                |
Temperature sensors ---- 1-Wire / sensor bus ---+--> ESP32-S2 Mini
                                                |      - sensor acquisition
Future sensors -------- provider interfaces ----+      - validation/filtering
                                                       - persistent counters
                                                       - diagnostics
                                                          |
                                                          +-- I2C --> irrigation controller
                                                          +-- MQTT --> Zmartify Edge
                                                          +-- HTTP --> diagnostics/provisioning
```

## 3. Responsibility boundaries

### Water Sensor node

- Own raw sensor acquisition and sensor-specific drivers.
- Own raw flow pulse counters and accumulated volume.
- Apply calibration, filtering and validity checks.
- Maintain a coherent, versioned sensor snapshot.
- Persist configuration and flow checkpoints.
- Publish telemetry and diagnostics.
- Continue acquisition during communication outages.

### Irrigation controller

- Own valves, pumps and irrigation sequencing.
- Consume measurements over I2C when local decisions require them.
- Implement local actuator interlocks and fail-safe behavior.
- Never depend exclusively on Wi-Fi or the Edge server for safety-critical control.

### Zmartify Edge Server

- Discover and register the node and its capabilities.
- Ingest and store historical measurements.
- Provide dashboards, alerts and automation.
- Manage desired configuration where supported.
- Track firmware, protocol and health information.

## 4. Sensor model

The firmware shall use a sensor registry rather than a single flow-channel-only model.

Every sensor has common metadata:

| Field | Description |
|---|---|
| `sensor_id` | Stable identifier within the device |
| `type` | Versioned sensor type such as `flow`, `temperature` or `soil_moisture` |
| `name` | Optional bounded friendly name |
| `enabled` | Whether acquisition is enabled |
| `present` | Whether the physical sensor is detected |
| `valid` | Whether the latest value is valid |
| `sample_age_ms` | Age of the latest valid sample |
| `fault_flags` | Sensor-specific and common faults |

Sensor-specific records extend this metadata.

### Flow sensor record

- monotonic pulse count;
- pulses per liter;
- accumulated volume in milliliters;
- current flow in milliliters per minute;
- last-pulse age.

### Temperature sensor record

- stable hardware identifier, such as a DS18B20 ROM code;
- temperature in milli-degrees Celsius;
- sample age;
- disconnected, CRC and range fault state.

### Future soil-moisture record

The initial architecture reserves this type but does not define a final electrical interface. A future record should expose raw reading, calibrated moisture percentage or permille, sample age and calibration profile identifier.

## 5. Authoritative state

The ESP32-S2 Mini is authoritative for measurements acquired by the node, including:

- flow pulse counters and accumulated volume;
- sensor identity and presence;
- calibration and enabled state;
- latest validated values and timestamps;
- sensor and device health.

I2C and MQTT are readout mechanisms for the same internal snapshot. They must not maintain independent counters or recalculate sensor values differently.

## 6. Availability requirements

- Pulse counting must continue during temperature conversion and bus scans.
- Sensor acquisition must continue during Wi-Fi reconnection.
- MQTT publication failure must never block acquisition.
- Flash writes must never occur in an interrupt handler.
- I2C callbacks must return prebuilt data without slow sensor access.
- A restart must restore the latest persisted flow totals.
- Missing or stale values must be represented as invalid or stale, never silently converted to zero.

## 7. Initial temperature design

The preferred initial temperature interface is externally powered, three-wire DS18B20 on 3.3 V 1-Wire buses.

Requirements:

- use stable ROM IDs as sensor identity;
- avoid parasite power for field installations;
- schedule non-blocking conversions;
- validate sensor CRC and plausible range;
- make bus topology and maximum cable length explicit in the hardware design;
- do not use a remote temperature measurement as the only physical protection for a safety-critical pump or heater.

## 8. Communication strategy

### I2C

Provides deterministic local access by the irrigation controller. The binary protocol exposes capabilities and typed sensor records. See [protocol-i2c.md](protocol-i2c.md).

### MQTT

Primary integration with Zmartify Edge. MQTT publishes retained identity/capabilities and typed telemetry. See [protocol-mqtt.md](protocol-mqtt.md).

### HTTP

A lightweight local interface may provide health, status, provisioning, metrics and OTA endpoints. HTTP is administrative, not the primary telemetry transport.

## 9. Security and safety

- Do not expose the device directly to the public internet.
- Protect Wi-Fi and MQTT credentials.
- Prefer authenticated MQTT and TLS.
- Validate all remotely supplied configuration.
- Validate OTA image integrity and support rollback where possible.
- Keep safety-critical actuator protection local to the actuator controller or dedicated hardware.

## 10. Evolution rules

New sensor types shall be added through new provider implementations, typed records, capability bits and versioned protocol additions. Existing field meanings and units must not change.
