# I2C Protocol

## 1. Scope

This protocol exposes the latest Zmartify Water Sensor snapshot to the local irrigation controller. The ESP32-S2 Mini operates as an I2C slave.

Default address: `0x32`, configurable and persisted.

## 2. Design rules

- Binary, compact and versioned.
- Little-endian fixed-width integers.
- Explicit message and record lengths.
- No compiler struct transmitted directly.
- Reads return coherent prebuilt snapshots.
- Unknown commands cannot change device state.
- Sensor acquisition and flow counting continue without an I2C master.
- Missing or stale values are flagged rather than encoded as valid zero values.

## 3. Transaction model

The master writes a one-byte command and then reads the response.

| Command | Name | Response |
|---:|---|---|
| `0x00` | `GET_IDENTITY` | Device, protocol and capability identity |
| `0x01` | `GET_STATUS` | Device status and provider summary |
| `0x10` | `GET_SENSOR_DIRECTORY` | Available typed sensor records |
| `0x11` | `GET_ALL_SENSORS` | Current records for all sensors |
| `0x12` | `GET_SENSOR` | One sensor selected by ID |
| `0x20` | `GET_DIAGNOSTICS` | Runtime and provider diagnostics |

Configuration writes are deferred until read-only access is stable and tested.

## 4. Common frame

| Offset | Size | Field |
|---:|---:|---|
| 0 | 1 | Protocol major version |
| 1 | 1 | Message type or command echo |
| 2 | 2 | Payload length |
| 4 | 4 | Snapshot sequence number |
| 8 | N | Payload |
| 8+N | 2 | CRC-16 over header and payload |

Recommended CRC: CRC-16/CCITT-FALSE. The implementation shall include shared test vectors.

## 5. Capabilities

Identity includes a capability bitmask. Initial capabilities include:

- flow measurement;
- accumulated volume;
- temperature measurement;
- typed sensor directory;
- MQTT, HTTP and OTA support.

Future capabilities may include soil moisture, pressure, leak detection and tank level.

## 6. Sensor directory record

Each directory entry identifies one stable logical sensor:

| Field | Type | Description |
|---|---|---|
| Sensor ID | uint16 | Stable ID within device |
| Sensor type | uint16 | Versioned type code |
| Flags | uint16 | Enabled, present and readable flags |
| Unit code | uint16 | Explicit unit identifier |
| Hardware ID | uint64 | ROM/address when available, otherwise zero |

Initial sensor type codes:

| Code | Type |
|---:|---|
| `0x0001` | Flow |
| `0x0002` | Temperature |
| `0x0003` | Soil moisture, reserved |
| `0x0004` | Pressure, reserved |
| `0x0005` | Leak/contact, reserved |

Codes and unit identifiers must be maintained in shared protocol definitions.

## 7. Common sensor record header

Every typed measurement record begins with:

| Field | Type | Description |
|---|---|---|
| Record length | uint16 | Allows skipping unknown record types |
| Sensor ID | uint16 | Stable logical ID |
| Sensor type | uint16 | Type code |
| Flags | uint16 | Enabled, present, valid, stale and activity flags |
| Fault flags | uint16 | Common or sensor-specific faults |
| Reserved | uint16 | Zero in protocol v1 |
| Sample age | uint32 | Milliseconds, saturating at `UINT32_MAX` |

Unknown record types can be skipped using `record_length`.

## 8. Flow record payload

Following the common header:

| Field | Type | Unit |
|---|---|---|
| Pulse count total | uint64 | pulses |
| Volume total | uint64 | milliliters |
| Current flow | uint32 | milliliters/minute |
| Last pulse age | uint32 | milliseconds |
| Calibration | uint32 | 0.001 pulse/L |

Four logical flow records are supported from the first revision.

## 9. Temperature record payload

Following the common header:

| Field | Type | Unit/description |
|---|---|---|
| Hardware ID | uint64 | DS18B20 ROM code or other stable address |
| Temperature | int32 | milli-degrees Celsius |
| Applied offset | int32 | milli-degrees Celsius |

The valid flag must be clear when disconnected, CRC-invalid, timed out or outside the accepted range. Consumers must inspect flags before using the numeric value.

## 10. Snapshot consistency

The acquisition tasks create typed records and an encoded response buffer outside the I2C callback. The callback only selects and copies bounded data.

It must not calculate flow, start temperature conversions, read flash, access Wi-Fi, allocate dynamic memory or wait indefinitely.

## 11. Versioning

- Increment major version for incompatible frame changes.
- Increment minor version for backward-compatible commands, record types or fields.
- Keep existing type codes, units and meanings stable.
- Add new record types instead of repurposing old fields.
- Record lengths allow older consumers to skip newer types.

## 12. Validation

Shared tests shall include flow and temperature records, invalid/stale sensors, unknown record skipping, counter values above 32 bits, signed temperatures below zero, maximum lengths and expected CRC values.
