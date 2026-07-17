# Firmware Architecture

## 1. Objectives

The firmware shall count pulses reliably while I2C, Wi-Fi, MQTT, HTTP, persistence and OTA activities occur concurrently.

Implementation should use Arduino-ESP32 or ESP-IDF behind narrow abstractions. Measurement-domain logic must be testable without ESP32 hardware.

## 2. Suggested repository structure

```text
firmware/
  include/
    app_config.h
    flow_channel.h
    measurement_snapshot.h
    protocol_i2c.h
    protocol_mqtt.h
  src/
    main.cpp
    flow_channel.cpp
    measurement_service.cpp
    persistence_service.cpp
    i2c_service.cpp
    wifi_service.cpp
    mqtt_service.cpp
    http_service.cpp
    diagnostics.cpp
  test/
    test_flow_calculation.cpp
    test_snapshot_serialization.cpp
    test_i2c_encoding.cpp
```

## 3. Components

### PulseCapture

- Configures four GPIO inputs.
- Registers interrupt handlers.
- Maintains raw monotonic counters and last-pulse timestamps.
- Performs no networking, logging, allocation or flash access in interrupts.

### MeasurementService

- Periodically takes an atomic copy of raw counters.
- Calculates pulse deltas and flow rates.
- Applies calibration and filtering.
- Updates an immutable or lock-protected `MeasurementSnapshot`.
- Detects timeout and implausible pulse-rate faults.

### PersistenceService

- Loads configuration and counter checkpoints from NVS.
- Saves only when thresholds require it.
- Uses versioned records and validates stored data.
- Reports storage faults without stopping pulse capture.

### I2CService

- Implements the slave protocol.
- Returns a prebuilt binary snapshot.
- Keeps callbacks bounded and allocation-free.
- Never performs measurement calculations inside an I2C callback.

### NetworkService

- Manages Wi-Fi connection and reconnection.
- Exposes connectivity state to diagnostics.
- Must never block measurement tasks waiting for a network.

### MQTTService

- Publishes retained availability and identity.
- Publishes periodic telemetry and state changes.
- Subscribes to versioned configuration topics if enabled.
- Validates all incoming commands.

### HTTPService

- Provides health and status endpoints.
- May provide provisioning and OTA endpoints.
- Reads the current snapshot rather than calculating measurements.

## 4. Interrupt design

Each channel may use a dedicated ISR or an ISR with channel context, depending on framework support.

ISR responsibilities:

```cpp
counter[channel]++;
lastPulseMicros[channel] = currentMicros;
```

The implementation shall consider:

- atomic access to 64-bit counters;
- interrupt-safe timestamp access;
- switch bounce or noise rejection;
- minimum accepted interval between pulses;
- timer wrap-safe unsigned arithmetic.

## 5. Measurement calculation

For each sampling interval:

```text
delta_pulses = current_count - previous_count
delta_liters = delta_pulses / pulses_per_liter
flow_l_min = delta_liters * 60000 / delta_time_ms
```

A low-flow mode should derive flow from the interval between recent pulses when window-based calculation becomes too coarse. The final algorithm may blend interval and window measurements.

Filtering must not change the total volume. Filtering applies only to displayed/published instantaneous flow.

## 6. Snapshot model

All external interfaces consume a coherent snapshot.

```cpp
struct ChannelSnapshot {
    bool enabled;
    uint64_t pulseCountTotal;
    float pulsesPerLiter;
    double volumeTotalLiters;
    float flowLitersPerMinute;
    uint32_t lastPulseAgeMs;
    uint16_t faultFlags;
};

struct MeasurementSnapshot {
    uint32_t sequence;
    uint32_t uptimeSeconds;
    uint8_t enabledMask;
    ChannelSnapshot channels[4];
};
```

The actual binary I2C representation must use fixed-width integers and explicit byte order. Do not transmit compiler structs directly.

## 7. Concurrency

Recommended execution model:

| Context | Typical period | Responsibility |
|---|---:|---|
| GPIO ISR | per pulse | Increment counter and timestamp |
| Measurement task | 100–1000 ms | Calculate and publish snapshot |
| I2C callback/task | on request | Copy encoded snapshot |
| MQTT task | 1–60 s/event | Publish snapshot and health |
| Persistence task | event/5 min | Save checkpoints/configuration |
| Diagnostics task | 1–10 s | Health and fault evaluation |

Use critical sections only for short counter copies. Do not hold locks while performing network or storage I/O.

## 8. Persistence

Persist at least:

- configuration schema version;
- device identifier and friendly name;
- enabled-channel mask;
- calibration per channel;
- accumulated pulse checkpoints;
- MQTT endpoint and credentials reference;
- publication intervals;
- I2C address;
- last known firmware migration version.

Suggested checkpoint triggers:

- every five minutes;
- after a configurable volume increment;
- when flow has stopped after activity;
- before a controlled restart or OTA update.

Use dual records, generation numbers or checksums to survive interrupted writes.

## 9. Configuration

Configuration changes shall be validated, applied atomically and persisted only after successful validation.

Constraints include:

- exactly four logical channels;
- `pulses_per_liter > 0` for enabled channels;
- bounded publish and measurement intervals;
- valid I2C address range;
- bounded strings;
- no secrets in logs or telemetry.

## 10. Fault flags

Suggested channel flags:

```text
0x0001 SENSOR_DISABLED
0x0002 NO_RECENT_PULSE
0x0004 PULSE_RATE_TOO_HIGH
0x0008 INPUT_STUCK_LOW
0x0010 INPUT_STUCK_HIGH
0x0020 CALIBRATION_INVALID
```

Suggested device flags:

```text
0x0001 STORAGE_ERROR
0x0002 WIFI_DISCONNECTED
0x0004 MQTT_DISCONNECTED
0x0008 CONFIGURATION_INVALID
0x0010 CLOCK_NOT_SYNCHRONIZED
0x0020 CHECKPOINT_PENDING
```

Connectivity states are normally health indicators, not fatal measurement errors.

## 11. Logging

Use structured log messages with level, component and event identifier. Never log Wi-Fi passwords, MQTT passwords or access tokens.

Rate-limit repeated connection errors and sensor warnings.

## 12. Testing

Unit tests shall cover:

- flow calculation at zero, low and high rates;
- counter wrap/delta logic;
- timer wrap handling;
- calibration validation;
- snapshot consistency;
- I2C byte encoding and CRC;
- MQTT payload schema;
- persistence migration and corrupted records;
- disabled and absent channels.

Hardware tests shall inject known pulse trains while Wi-Fi reconnects, MQTT is unavailable, I2C is polled and flash checkpoints occur.
