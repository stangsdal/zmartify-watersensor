# System Architecture

## 1. Purpose

`zmartify-flowmeter` is a reusable four-channel pulse counter and flow-measurement node based on an ESP32-S2 Mini V1.0.0.

The node shall:

- count pulses from up to four flow meters;
- calculate current flow and accumulated volume;
- preserve totals across restarts;
- expose local measurements over I2C;
- publish measurements over Wi-Fi to Zmartify Edge;
- remain operational when Wi-Fi, MQTT, the Edge server or the I2C master is unavailable.

## 2. System context

```text
Flow meter 1..4
      |
      v
Input protection and 5 V to 3.3 V adaptation
      |
      v
ESP32-S2 Mini
  - pulse counting
  - flow calculation
  - persistent totals
  - diagnostics
      |
      +---- I2C ---- local controller
      |
      +---- Wi-Fi / MQTT ---- Zmartify Edge Server
      |
      +---- HTTP ---- local diagnostics and provisioning
```

## 3. Responsibilities

### Flowmeter node

- Own all raw pulse counters.
- Apply per-channel calibration.
- Calculate current flow and total volume.
- Persist configuration and checkpoints.
- Expose consistent measurement snapshots.
- Detect channel and system faults.
- Reconnect automatically after network failures.

### Zmartify Edge Server

- Discover and register the node.
- Ingest telemetry.
- Store historical measurements.
- Provide dashboards, alerts and automation.
- Manage desired configuration where supported.
- Track firmware, protocol and health information.

The Edge server shall not depend on receiving every individual pulse. It consumes calculated measurements and monotonic counters.

## 4. Authoritative state

The ESP32-S2 Mini is authoritative for:

- `pulse_count_total`;
- `volume_total_l`;
- channel calibration;
- channel enabled state;
- last-pulse timestamp;
- current calculated flow;
- device health.

I2C and MQTT are readout mechanisms for the same internal snapshot. They must not maintain separate counters.

## 5. Data model

Each of four channels has:

| Field | Type | Unit | Description |
|---|---:|---|---|
| `enabled` | boolean | — | Whether the channel participates in measurement |
| `pulse_count_total` | uint64 | pulses | Monotonic pulse count |
| `pulses_per_liter` | float | pulses/L | Calibration factor |
| `volume_total_l` | double/derived | L | Accumulated volume |
| `flow_l_min` | float | L/min | Filtered current flow |
| `last_pulse_age_ms` | uint32 | ms | Time since last valid pulse |
| `fault_flags` | bit field | — | Channel diagnostics |

The device snapshot additionally contains:

- device identifier;
- firmware version;
- schema/protocol versions;
- boot identifier;
- uptime;
- Wi-Fi and MQTT state;
- persistence state;
- enabled-channel bitmask.

## 6. Availability requirements

- Pulse counting must continue during Wi-Fi reconnection.
- Pulse counting must continue while I2C transactions occur.
- MQTT publication failure must never block measurement.
- Flash writes must never occur in an interrupt handler.
- A software restart must restore the latest persisted totals.
- A 32-bit millisecond timer wrap must not invalidate flow calculation.

## 7. Communication strategy

### I2C

Used for deterministic local access by another controller. The interface is binary, compact and versioned. See [protocol-i2c.md](protocol-i2c.md).

### MQTT

Primary integration with Zmartify Edge. MQTT carries retained device identity/status and periodic telemetry. See [protocol-mqtt.md](protocol-mqtt.md).

### HTTP

A lightweight local interface may provide:

- `GET /api/v1/health`;
- `GET /api/v1/status`;
- local provisioning/configuration;
- optional metrics and firmware update endpoints.

HTTP is diagnostic and administrative. It is not the primary telemetry transport.

## 8. Security boundaries

- Do not expose the device directly to the public internet.
- Store Wi-Fi and MQTT credentials in protected persistent storage.
- Prefer authenticated MQTT and TLS when supported by the deployment.
- Validate all remotely supplied configuration values.
- OTA updates must validate image integrity and support rollback where the platform permits it.

## 9. Initial and future scope

Initial deployment uses two flow meters. The complete design supports four channels without protocol or architecture changes.

Potential future extensions include pressure sensors, temperature sensors, leak detectors and valve control. New capabilities must use new versioned fields or topics and must not change the meaning of existing fields.
