# MQTT and Zmartify Edge Protocol

## 1. Purpose

MQTT is the primary Wi-Fi integration between the Zmartify Water Sensor node and Zmartify Edge. The protocol supports discovery, typed telemetry, health monitoring and future remote configuration.

## 2. Topic root

```text
zmartify/watersensor/<device_id>/
```

`device_id` must be stable, unique and safe for use in a topic. It must not depend only on a user-editable friendly name.

## 3. Topics

| Topic suffix | Retained | Direction | Description |
|---|---:|---|---|
| `availability` | yes | device -> Edge | `online` or `offline` |
| `identity` | yes | device -> Edge | Model, firmware and capabilities |
| `state` | yes | device -> Edge | Latest complete sensor snapshot |
| `telemetry` | no | device -> Edge | Periodic changed/current measurements |
| `health` | no/yes | device -> Edge | Runtime and provider health |
| `event` | no | device -> Edge | Fault and lifecycle events |
| `config/set` | no | Edge -> device | Desired configuration |
| `config/state` | yes | device -> Edge | Applied configuration |
| `command` | no | Edge -> device | Versioned administrative commands |
| `response` | no | device -> Edge | Command acknowledgement/result |

## 4. Availability

Configure Last Will and Testament on:

```text
zmartify/watersensor/<device_id>/availability
```

Use retained `offline` with QoS 1 and publish retained `online` after connection.

## 5. Identity payload

```json
{
  "schema": "zmartify.watersensor.identity.v1",
  "device_id": "watersensor-a1b2c3",
  "name": "House water sensor",
  "model": "esp32-s2-mini-v1",
  "firmware_version": "0.2.0",
  "protocol_version": 1,
  "capabilities": [
    "flow",
    "total_volume",
    "temperature",
    "typed_sensors",
    "i2c",
    "http",
    "ota"
  ],
  "sensor_types": ["flow", "temperature"]
}
```

Future firmware may add `soil_moisture`, `pressure`, `leak` or other capabilities without changing existing semantics.

## 6. Telemetry payload

```json
{
  "schema": "zmartify.watersensor.telemetry.v1",
  "device_id": "watersensor-a1b2c3",
  "sequence": 1842,
  "boot_id": "8d8d48f1",
  "timestamp": "2026-07-17T12:00:00Z",
  "uptime_s": 83214,
  "sensors": [
    {
      "sensor_id": "flow-1",
      "type": "flow",
      "enabled": true,
      "present": true,
      "valid": true,
      "sample_age_ms": 120,
      "pulse_count_total": 987654,
      "volume_total_l": 1243.8,
      "flow_l_min": 6.4,
      "last_pulse_age_ms": 120,
      "faults": []
    },
    {
      "sensor_id": "temperature-water-inlet",
      "type": "temperature",
      "hardware_id": "28ff64a29116047c",
      "enabled": true,
      "present": true,
      "valid": true,
      "sample_age_ms": 814,
      "temperature_c": 12.375,
      "faults": []
    }
  ]
}
```

Every record must include a stable `sensor_id`, explicit `type`, validity and age. Missing, disconnected or stale sensors remain represented with `valid: false`; they are not reported as valid zero readings.

## 7. Sensor type conventions

### Flow

- `pulse_count_total`: monotonic counter.
- `volume_total_l`: monotonic accumulated volume.
- `flow_l_min`: gauge.
- `last_pulse_age_ms`: age indicator.

### Temperature

- `temperature_c`: signed Celsius gauge.
- `hardware_id`: stable ROM/address when available.
- optional `offset_c` may be included in configuration state, not required in every telemetry message.

### Future soil moisture

A later schema extension may add:

- `raw_value`;
- `moisture_percent` or `moisture_permille`;
- `calibration_profile`;
- probe and supply diagnostics.

The final units and validity rules must be defined before implementation.

## 8. Publication policy

Recommended defaults:

- retained complete state on startup and material state changes;
- active flow telemetry every 10 seconds;
- idle telemetry every 60 seconds;
- temperature telemetry on meaningful change and at least every 60 seconds;
- immediate events for raised or cleared faults;
- health every 60 seconds;
- identity after every successful MQTT connection.

Intervals are configurable and bounded. Acquisition cadence is independent of publication cadence.

## 9. Zmartify Edge ingestion

The Edge adapter shall:

1. register a device and its capabilities from `identity`;
2. register sensors by stable `sensor_id` and `type`;
3. validate `schema` before parsing;
4. store monotonic counters separately from gauges;
5. preserve units and validity state;
6. use `sequence` and `boot_id` for ordering and restart detection;
7. treat missing telemetry as a connectivity problem, not a zero measurement;
8. reject or quarantine unknown incompatible schema versions;
9. allow newly introduced sensor types to be ignored safely by older consumers.

Suggested metric families:

```text
zmartify_watersensor_flow_l_min
zmartify_watersensor_volume_total_l
zmartify_watersensor_pulse_count_total
zmartify_watersensor_temperature_c
zmartify_watersensor_sensor_valid
zmartify_watersensor_sensor_fault
zmartify_watersensor_online
```

Labels should use bounded stable identifiers and sensor type. Avoid unbounded friendly text as metric labels.

## 10. Configuration

A future configuration payload uses a request ID and typed provider settings.

```json
{
  "schema": "zmartify.watersensor.config.v1",
  "request_id": "01J2ABCDEF",
  "flow_sensors": [
    {"sensor_id": "flow-1", "enabled": true, "pulses_per_liter": 450.0}
  ],
  "temperature_sensors": [
    {
      "sensor_id": "temperature-water-inlet",
      "hardware_id": "28ff64a29116047c",
      "enabled": true,
      "offset_c": 0.0
    }
  ],
  "telemetry_active_interval_s": 10,
  "telemetry_idle_interval_s": 60
}
```

Configuration must be validated, atomically applied, persisted and acknowledged. Invalid configuration leaves the previous configuration active.

## 11. Health and security

Health includes uptime, reset reason, Wi-Fi RSSI, reconnect counts, heap, provider status, bus errors, storage status, checkpoint age and firmware/protocol versions.

Use authenticated MQTT, prefer TLS, never publish secrets, restrict credentials to the device topic subtree, validate payload size/depth/ranges and reject unknown command schemas.

## 12. Schema evolution

Compatible fields and new sensor record types may be added to v1 when older consumers can ignore them. Existing field meanings and units remain immutable. Breaking changes require a new schema name such as `v2`.
