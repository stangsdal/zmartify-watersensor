# Copilot Implementation Instructions

## 1. Role

Use this document as the implementation contract when generating or reviewing code for `zmartify-flowmeter`.

The design documents in `docs/` are normative. When code and documentation disagree, do not silently invent behavior; update the design or clearly mark the discrepancy.

## 2. Target

- Controller: ESP32-S2 Mini V1.0.0.
- Language: modern C++ supported by the selected ESP32 toolchain.
- Initial channels: 2.
- Supported channels: exactly 4 from the first firmware version.
- Interfaces: pulse GPIO, I2C slave, Wi-Fi, MQTT and diagnostic HTTP.

Do not hard-code assumptions that only two channels exist.

## 3. Architectural rules

1. The ESP32-S2 is the authoritative pulse counter.
2. Pulse capture must work without Wi-Fi, MQTT, HTTP or I2C.
3. All external interfaces consume the same coherent measurement snapshot.
4. Interrupt handlers only update counters and timestamps.
5. Do not allocate memory, log, access flash or call network code from an ISR.
6. Do not transmit in-memory C++ structs as protocol frames.
7. Use explicit fixed-width integer encoding and documented byte order.
8. Keep measurement-domain calculations independent of hardware APIs and unit-testable.
9. Use non-blocking state machines or bounded tasks for connectivity.
10. Never erase or reset accumulated totals merely because a consumer has read them.

## 4. Coding conventions

- Prefer `constexpr` over macros for constants.
- Use `std::array<T, 4>` for four-channel collections where toolchain support permits it.
- Use fixed-width types such as `uint32_t` and `uint64_t` for protocol and counter fields.
- Use descriptive names with units, for example `flowLitersPerMinute` and `lastPulseAgeMs`.
- Keep functions small and single-purpose.
- Avoid global mutable state except minimal ISR-visible state hidden behind a component.
- Avoid dynamic allocation in steady-state firmware paths.
- Validate every external input and configuration value.
- Never log credentials, tokens or complete secret-bearing configuration objects.
- Include clear error handling; do not ignore return values from storage or network operations.

## 5. Required abstractions

Generate interfaces equivalent to:

```cpp
class IPulseCapture;
class IMeasurementService;
class IPersistenceStore;
class II2cTransport;
class IMqttTransport;
class IClock;
```

Concrete ESP32 implementations may depend on Arduino-ESP32 or ESP-IDF. Domain tests should use fakes.

Do not over-engineer with deep inheritance. Prefer composition and small interfaces.

## 6. Measurement requirements

For every channel maintain:

- enabled state;
- total 64-bit pulse count;
- calibration in pulses per liter;
- accumulated volume;
- current filtered flow;
- last-pulse age;
- channel fault flags.

Filtering may change displayed instantaneous flow but must never change total volume.

Handle:

- zero flow;
- low flow where pulses are far apart;
- high flow;
- timer wrap;
- restart restoration;
- absent and disabled channels;
- noisy pulses or a configurable minimum pulse interval.

## 7. Concurrency rules

- Copy ISR counters inside the shortest possible critical section.
- Perform floating-point calculations outside critical sections.
- Do not hold locks during MQTT, HTTP, I2C or NVS operations.
- Build or encode snapshots before time-critical callbacks when possible.
- I2C callbacks must be bounded and allocation-free.
- Network reconnection loops must yield and use backoff.

## 8. Persistence rules

- Version every stored record.
- Validate records before use.
- Support safe defaults when no valid configuration exists.
- Minimize flash wear; never save for every pulse.
- Preserve monotonic totals across ordinary restarts.
- Use generation/checksum or equivalent protection against interrupted writes.
- Add explicit migration code when persistent schema changes.

## 9. I2C rules

Follow `docs/protocol-i2c.md`.

- Little-endian encoding.
- Fixed command identifiers.
- CRC according to the documented variant.
- Stable meanings for existing commands.
- Unit tests with raw hexadecimal test vectors.
- Malformed requests must not reset the device or alter counters.

Do not implement write/configuration commands until read-only snapshot access is tested.

## 10. MQTT and Edge rules

Follow `docs/protocol-mqtt.md`.

- Topic root uses stable `device_id`.
- Publish retained identity, state and availability.
- Configure an offline Last Will.
- Include schema name, sequence and boot identifier.
- Represent all four channels in complete state payloads.
- Treat totals as monotonic counters and flow as a gauge.
- Validate command schema, payload size and ranges.
- Keep MQTT optional for measurement operation.

## 11. HTTP rules

Initial endpoints:

```text
GET /api/v1/health
GET /api/v1/status
```

Responses read the current snapshot. Endpoints must not trigger measurement calculations or flash writes.

Do not expose the device directly to the public internet.

## 12. Testing requirements

For each implementation unit, generate tests before or with production code.

Minimum tests:

- pulse-to-volume conversion;
- flow calculation at multiple sample intervals;
- low-flow interval calculation;
- calibration validation;
- timer and counter wrap handling;
- snapshot atomicity/consistency;
- I2C encoding and CRC vectors;
- MQTT JSON schema examples;
- persistence recovery from invalid and interrupted records;
- all combinations of enabled channels;
- reconnect behavior that does not stop pulse capture.

Tests must not depend on real time when a fake clock can be injected.

## 13. Definition of done

A feature is not complete until:

- code builds for ESP32-S2;
- relevant unit tests pass;
- protocol changes are documented;
- no secrets are logged;
- measurement continues during simulated network outage;
- four-channel behavior is covered even when only two physical meters are installed;
- any new persistent or external schema has an explicit version.

## 14. First implementation milestones

1. Establish build system and board configuration.
2. Implement four-channel pulse capture with synthetic pulse tests.
3. Implement measurement calculation and snapshots.
4. Implement versioned persistence.
5. Implement read-only I2C protocol.
6. Implement Wi-Fi provisioning/reconnection.
7. Implement MQTT identity, availability and telemetry.
8. Implement health/status HTTP endpoints.
9. Add OTA and rollback strategy.
10. Run end-to-end pulse-loss and restart tests.
