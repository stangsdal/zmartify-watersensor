# I2C Protocol

## 1. Scope

This protocol exposes the latest four-channel measurement snapshot to a local Zmartify controller. The ESP32-S2 Mini operates as an I2C slave.

Default address: `0x32`.

The address shall be configurable and stored persistently.

## 2. Design rules

- Binary, compact and versioned.
- Little-endian integers.
- Fixed-width integer fields.
- No compiler struct is transmitted directly.
- Reads return a coherent prebuilt snapshot.
- Unknown commands shall not change device state.
- The protocol must remain functional without Wi-Fi.

## 3. Transaction model

The master writes a one-byte command/register identifier and then reads the expected response length.

Initial commands:

| Command | Name | Response |
|---:|---|---|
| `0x00` | `GET_IDENTITY` | Device and protocol identity |
| `0x01` | `GET_STATUS` | Device status and enabled mask |
| `0x10` | `GET_ALL_CHANNELS` | Four-channel measurement snapshot |
| `0x11`–`0x14` | `GET_CHANNEL_1..4` | One channel snapshot |
| `0x20` | `GET_DIAGNOSTICS` | Runtime diagnostics |

Configuration writes should be added only after read-only measurement access is stable and tested.

## 4. Common frame

Responses use:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 1 | Protocol major version |
| 1 | 1 | Message type/command echo |
| 2 | 2 | Payload length, little-endian |
| 4 | 4 | Snapshot sequence number |
| 8 | N | Payload |
| 8+N | 2 | CRC-16 over header and payload |

CRC polynomial and initialization must be fixed in implementation and test vectors. Recommended starting choice: CRC-16/CCITT-FALSE.

## 5. Channel encoding

A channel payload uses integer-scaled values:

| Field | Type | Scale | Description |
|---|---|---:|---|
| Channel index | uint8 | 1 | `0..3` |
| Flags | uint8 | 1 | Enabled/present/activity flags |
| Fault flags | uint16 | 1 | Channel faults |
| Pulse count total | uint64 | 1 pulse | Monotonic count |
| Volume total | uint64 | 1 mL | Accumulated volume |
| Current flow | uint32 | 1 mL/min | Filtered current flow |
| Last pulse age | uint32 | 1 ms | Saturates at `UINT32_MAX` |
| Calibration | uint32 | 0.001 pulse/L | Pulses per liter multiplied by 1000 |

Using scaled integers avoids floating-point representation differences between controllers.

## 6. `GET_ALL_CHANNELS`

Payload:

| Field | Type | Description |
|---|---|---|
| Enabled mask | uint8 | Bits 0–3 correspond to channels 1–4 |
| Active mask | uint8 | Channels with recent flow |
| Device fault flags | uint16 | Global diagnostic flags |
| Uptime | uint32 | Seconds |
| Channel records | 4 × channel payload | Fixed four records |

All four records are returned even when only two channels are enabled.

## 7. Identity response

`GET_IDENTITY` should contain:

- protocol major and minor version;
- firmware semantic version;
- hardware model identifier;
- stable device identifier;
- supported channel count;
- capability bitmask.

Strings must have explicit maximum lengths or be represented by fixed identifiers.

## 8. Snapshot consistency

The measurement task creates an encoded response buffer or immutable snapshot. The I2C handler copies from that state.

The handler must not:

- calculate flow;
- read flash;
- access Wi-Fi;
- wait for another task for an unbounded time;
- allocate dynamic memory.

The sequence number increments whenever a new measurement snapshot is published internally. A master can detect repeated or missed snapshots with this field.

## 9. Error behavior

For an unsupported command, return an error frame with an error code such as:

| Code | Meaning |
|---:|---|
| `0x01` | Unsupported command |
| `0x02` | Invalid request length |
| `0x03` | Busy; retry later |
| `0x04` | Configuration invalid |
| `0x05` | Internal error |

The device shall never reset or corrupt counters due to a malformed I2C request.

## 10. Versioning

- Increment the major version for incompatible frame or field changes.
- Increment the minor version for backward-compatible additions.
- Keep existing command meanings stable.
- Add new commands rather than repurposing old ones.

## 11. Validation

Provide shared protocol test vectors containing:

- raw hexadecimal frame;
- decoded field values;
- expected CRC;
- minimum and maximum values;
- disabled-channel example;
- counter values above 32 bits.

The firmware and Edge/local-controller implementation should run the same test vectors in CI.
