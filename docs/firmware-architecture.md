# Firmware Architecture

## 1. Objectives

The firmware shall acquire heterogeneous sensors reliably while I2C, Wi-Fi, MQTT, HTTP, persistence and OTA activities occur concurrently.

Measurement-domain logic must be testable without ESP32 hardware. Hardware-specific drivers shall be hidden behind narrow interfaces.

## 2. Suggested repository structure

```text
firmware/
  include/
    app_config.h
    sensor_provider.h
    sensor_registry.h
    measurement_snapshot.h
    flow_provider.h
    temperature_provider.h
    protocol_i2c.h
    protocol_mqtt.h
  src/
    main.cpp
    sensor_registry.cpp
    flow_provider.cpp
    temperature_provider.cpp
    measurement_service.cpp
    persistence_service.cpp
    i2c_service.cpp
    wifi_service.cpp
    mqtt_service.cpp
    http_service.cpp
    diagnostics.cpp
  test/
    test_flow_calculation.cpp
    test_temperature_validation.cpp
    test_sensor_registry.cpp
    test_snapshot_serialization.cpp
    test_i2c_encoding.cpp
    test_mqtt_schema.cpp
```

## 3. Core abstractions

### SensorProvider

Every sensor family implements a provider interface responsible for:

- initialization and capability reporting;
- non-blocking acquisition scheduling;
- validation and fault reporting;
- configuration validation;
- publishing typed records into the common snapshot;
- optional persistent state.

A provider failure must not stop unrelated providers.

### SensorRegistry

The registry owns stable sensor identifiers and the latest typed records. It provides coherent immutable or lock-protected snapshots to I2C, MQTT and HTTP.

### FlowProvider

- Configures four GPIO pulse inputs.
- Maintains monotonic counters and last-pulse timestamps in interrupt-safe state.
- Calculates flow and accumulated volume.
- Applies calibration and filtering.
- Persists counter checkpoints.

Interrupt handlers may only update bounded in-memory state. They must not perform networking, logging, allocation, sensor-bus access or flash writes.

### TemperatureProvider

The initial implementation targets externally powered DS18B20 sensors.

Responsibilities:

- discover sensors and read stable ROM IDs;
- map configured ROM IDs to logical sensor IDs;
- start conversions without blocking other work;
- collect results after the required conversion interval;
- validate CRC, range and presence;
- publish temperature in milli-degrees Celsius;
- retain the previous value as stale/invalid rather than replacing an error with zero.

### Future providers

Soil moisture, pressure, leak and level sensors shall use the same provider contract. New providers must not require changes to existing flow calculations or communication services.

## 4. Snapshot model

External interfaces consume one coherent snapshot.

```cpp
struct SensorCommon {
    uint16_t sensorId;
    uint16_t type;
    uint16_t flags;
    uint16_t faultFlags;
    uint32_t sampleAgeMs;
};

struct FlowRecord {
    SensorCommon common;
    uint64_t pulseCountTotal;
    uint64_t volumeTotalMilliliters;
    uint32_t flowMillilitersPerMinute;
    uint32_t lastPulseAgeMs;
};

struct TemperatureRecord {
    SensorCommon common;
    uint64_t hardwareId;
    int32_t temperatureMilliCelsius;
};
```

These are domain examples, not wire-format structs. I2C encoding must use explicit fields, lengths and byte order.

## 5. Execution model

| Context | Typical period | Responsibility |
|---|---:|---|
| GPIO ISR | per pulse | Update flow counter and timestamp |
| Flow calculation | 100–1000 ms | Calculate flow and totals |
| Temperature state machine | configurable | Start conversions and collect results |
| Snapshot service | 100–1000 ms/event | Publish coherent registry snapshot |
| I2C callback/task | on request | Copy prebuilt encoded data |
| MQTT task | 1–60 s/event | Publish state, telemetry and health |
| Persistence task | event/5 min | Save checkpoints and configuration |
| Diagnostics task | 1–10 s | Evaluate provider and device health |

Use short critical sections only for counter or snapshot copies. Never hold locks during network, sensor conversion or storage I/O.

## 6. Flow calculation

```text
delta_pulses = current_count - previous_count
delta_liters = delta_pulses / pulses_per_liter
flow_l_min = delta_liters * 60000 / delta_time_ms
```

Filtering applies only to instantaneous flow, never accumulated totals. A low-flow algorithm may use pulse intervals when window-based calculation becomes too coarse.

## 7. Temperature acquisition

Temperature conversion shall be implemented as a state machine:

1. request conversion for a bus or sensor set;
2. return control immediately;
3. wait using timestamps rather than blocking delays;
4. read scratchpads after conversion time;
5. validate CRC and range;
6. update records and schedule the next cycle.

The conversion cadence is configurable and typically much slower than flow sampling.

## 8. Persistence

Persist at least:

- configuration schema version;
- device identity and friendly name;
- enabled providers and sensors;
- flow calibration and pulse checkpoints;
- temperature ROM-to-logical-ID mapping and optional offsets;
- MQTT and publication settings;
- I2C address;
- firmware migration version.

Use versioned records with checksums or dual generations. Minimize flash writes.

## 9. Fault model

Common sensor faults include disabled, absent, stale, bus error, invalid configuration and out-of-range value.

Flow-specific faults include implausible pulse rate, stuck input and invalid calibration.

Temperature-specific faults include ROM mismatch, scratchpad CRC failure, conversion timeout, disconnected sensor and implausible temperature.

Connectivity faults are health indicators and must not be treated as measurement values.

## 10. Configuration rules

- sensor IDs are stable and unique within a device;
- type and unit semantics are immutable for a sensor ID;
- all numeric ranges are validated;
- configuration applies atomically;
- unknown provider configuration is rejected safely;
- secrets are never logged or published;
- removing a physical sensor does not silently reassign its identity to another sensor.

## 11. Testing

Unit tests shall cover flow calculations, timer wrap, temperature validation, provider isolation, registry consistency, serialization, persistence migration and invalid configuration.

Hardware tests shall run pulse trains and temperature conversions while Wi-Fi reconnects, MQTT is unavailable, I2C is polled and checkpoints occur. A broken 1-Wire bus must not stop flow counting.
