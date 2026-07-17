# MQTT and Zmartify Edge Protocol

## 1. Purpose

MQTT is the primary Wi-Fi integration between the flowmeter node and Zmartify Edge. The protocol is designed for discovery, telemetry, health monitoring and future remote configuration.

## 2. Topic root

```text
zmartify/flowmeter/<device_id>/
```

`device_id` must be stable, unique and safe for use in an MQTT topic. It must not depend only on a user-editable friendly name.

## 3. Topics

| Topic suffix | Retained | Direction | Description |
|---|---:|---|---|
| `availability` | yes | device → Edge | `online` or `offline` |
| `identity` | yes | device → Edge | Model, firmware and capabilities |
| `state` | yes | device → Edge | Latest complete measurement snapshot |
| `telemetry` | no | device → Edge | Periodic measurements |
| `health` | no/yes | device → Edge | Runtime and connection health |
| `event` | no | device → Edge | Fault and lifecycle events |
| `config/set` | no | Edge → device | Desired configuration |
| `config/state` | yes | device → Edge | Applied configuration |
| `command` | no | Edge → device | Versioned administrative commands |
| `response` | no | device → Edge | Command acknowledgement/result |

The initial implementation may omit remote writes while retaining the topic design.

## 4. Availability

Configure an MQTT Last Will and Testament:

```text
Topic:   zmartify/flowmeter/<device_id>/availability
Payload: offline
Retain:  true
QoS:     1
```

After connection, publish retained `online`.

## 5. Identity payload

Example:

```json
{
  "schema": "zmartify.flowmeter.identity.v1",
  "device_id": "flowmeter-a1b2c3",
  "name": "House water meter",
  "model": "esp32-s2-mini-v1",
  "firmware_version": "0.1.0",
  "protocol_version": 1,
  "channel_count": 4,
  "capabilities": ["flow", "total_volume", "i2c", "http", "ota"]
}
```

## 6. State and telemetry payload

Example:

```json
{
  "schema": "zmartify.flowmeter.telemetry.v1",
  "device_id": "flowmeter-a1b2c3",
  "sequence": 1842,
  "boot_id": "8d8d48f1",
  "timestamp": "2026-07-17T12:00:00Z",
  "uptime_s": 83214,
  "channels": [
    {
      "channel": 1,
      "enabled": true,
      "pulse_count_total": 987654,
      "volume_total_l": 1243.8,
      "flow_l_min": 6.4,
      "last_pulse_age_ms": 120,
      "faults": []
    },
    {
      "channel": 2,
      "enabled": true,
      "pulse_count_total": 12450,
      "volume_total_l": 18.6,
      "flow_l_min": 0.0,
      "last_pulse_age_ms": 4294967295,
      "faults": []
    },
    {
      "channel": 3,
      "enabled": false,
      "pulse_count_total": 0,
      "volume_total_l": 0.0,
      "flow_l_min": 0.0,
      "last_pulse_age_ms": 4294967295,
      "faults": ["sensor_disabled"]
    },
    {
      "channel": 4,
      "enabled": false,
      "pulse_count_total": 0,
      "volume_total_l": 0.0,
      "flow_l_min": 0.0,
      "last_pulse_age_ms": 4294967295,
      "faults": ["sensor_disabled"]
    }
  ]
}
```

All four channels should be represented to keep ingestion predictable.

## 7. Publication policy

Recommended defaults:

- complete retained state on startup and material state changes;
- telemetry every 10 seconds while flow is active;
- telemetry every 60 seconds while idle;
- immediate event for newly raised or cleared faults;
- health every 60 seconds;
- identity on every successful MQTT connection.

Publication intervals must be configurable and bounded.

## 8. QoS and retention

- Availability: QoS 1, retained.
- Identity: QoS 1, retained.
- State: QoS 1, retained.
- Telemetry: QoS 0 or 1 depending on deployment requirements.
- Events: QoS 1, not retained.
- Commands and responses: QoS 1.

The monotonic counters make duplicate delivery harmless when consumers are idempotent.

## 9. Zmartify Edge ingestion

The Edge adapter should:

1. Register a device from `identity`.
2. Track online state from `availability`.
3. Validate `schema` before parsing.
4. Store `pulse_count_total` and `volume_total_l` as monotonic counters.
5. Store `flow_l_min` as a gauge.
6. Use `sequence` and `boot_id` to identify restart and ordering behavior.
7. Treat missing telemetry as a connectivity problem, not as zero flow.
8. Generate leak and usage rules from stored measurements rather than altering raw totals.

Suggested metric names:

```text
zmartify_flowmeter_flow_l_min
zmartify_flowmeter_volume_total_l
zmartify_flowmeter_pulse_count_total
zmartify_flowmeter_last_pulse_age_ms
zmartify_flowmeter_channel_fault
zmartify_flowmeter_online
```

Labels should include stable `device_id` and channel number. Avoid unbounded user text as metric labels.

## 10. Remote configuration

A future `config/set` payload should include a request identifier and complete or explicitly patched configuration.

Example:

```json
{
  "schema": "zmartify.flowmeter.config.v1",
  "request_id": "01J2ABCDEF",
  "channels": [
    {"channel": 1, "enabled": true, "pulses_per_liter": 450.0},
    {"channel": 2, "enabled": true, "pulses_per_liter": 450.0}
  ],
  "telemetry_active_interval_s": 10,
  "telemetry_idle_interval_s": 60
}
```

The device must validate, atomically apply, persist and acknowledge configuration. Invalid configuration must leave the previous configuration active.

## 11. Health payload

Include at least:

- uptime;
- reset reason;
- Wi-Fi RSSI;
- MQTT reconnect count;
- free heap and minimum free heap;
- storage status;
- last successful checkpoint age;
- device fault flags;
- firmware and protocol versions.

## 12. Security

- Use authenticated MQTT.
- Prefer TLS with server certificate validation.
- Do not place secrets in topics or payloads.
- Restrict device credentials to its own topic subtree where possible.
- Validate payload size, JSON depth, schema and numeric ranges.
- Rate-limit command processing and reject unknown schema versions.

## 13. Schema evolution

The `schema` value identifies payload semantics. Compatible fields may be added to a v1 object, but existing field meaning and units must remain unchanged. Breaking changes require a new schema name such as `v2`.
