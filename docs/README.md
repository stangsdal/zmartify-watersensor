# Zmartify Flowmeter Documentation

This directory contains the design specification for the `zmartify-flowmeter` module.

## Documents

- [Architecture](architecture.md) — system boundaries, responsibilities and data flow.
- [Hardware](hardware.md) — ESP32-S2 Mini, four pulse inputs, level shifting and wiring.
- [Firmware Architecture](firmware-architecture.md) — components, tasks, concurrency and persistence.
- [I2C Protocol](protocol-i2c.md) — binary interface for a local Zmartify controller.
- [MQTT and Edge Protocol](protocol-mqtt.md) — Wi-Fi integration with Zmartify Edge.
- [Calibration](calibration.md) — channel calibration and verification procedure.
- [Copilot Instructions](copilot-instructions.md) — implementation rules for AI-assisted development.

## Status

The documents describe the initial four-channel design. Two channels may be populated in the first deployment, but hardware, firmware and protocols shall support four channels from the start.

## Design principles

1. Pulse counting is local and independent of Wi-Fi, MQTT and the Edge server.
2. The ESP32-S2 Mini is the authoritative source for pulse counts and accumulated volume.
3. I2C and MQTT expose snapshots of the same internal measurement state.
4. Network outages must not cause lost pulses or reset accumulated totals.
5. Protocols are versioned and designed for backward-compatible extension.
