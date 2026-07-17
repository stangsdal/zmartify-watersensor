# Zmartify Water Sensor Documentation

This directory contains the design specification for `zmartify-watersensor`, an ESP32-S2 based sensor node for water and irrigation installations.

## Documents

- [Architecture](architecture.md) — system boundaries, responsibilities, sensor model and data flow.
- [Hardware](hardware.md) — ESP32-S2 Mini, flow inputs, temperature buses and expansion interfaces.
- [Firmware Architecture](firmware-architecture.md) — components, tasks, concurrency and persistence.
- [I2C Protocol](protocol-i2c.md) — binary interface for the irrigation controller.
- [MQTT and Edge Protocol](protocol-mqtt.md) — Wi-Fi integration with Zmartify Edge.
- [Calibration](calibration.md) — flow-channel calibration and verification.
- [Copilot Instructions](copilot-instructions.md) — implementation rules for AI-assisted development.

## Initial scope

The first revision supports:

- four pulse-based flow-meter inputs, with two initially populated;
- temperature sensors, initially expected to use externally powered DS18B20 devices;
- I2C readout by the irrigation controller;
- Wi-Fi/MQTT telemetry to Zmartify Edge;
- persistent flow totals and versioned configuration.

## Expansion scope

The architecture must allow additional sensor providers without redesigning the core, including soil moisture, pressure, leak detection, tank level, conductivity and environmental sensors.

## Design principles

1. Sensor acquisition is local and independent of Wi-Fi, MQTT and the Edge server.
2. The ESP32-S2 Mini is authoritative for its raw measurements, calibration and accumulated flow totals.
3. The irrigation controller remains authoritative for valves, pumps and safety-critical actuator control.
4. I2C and MQTT expose coherent snapshots from the same internal sensor registry.
5. Network outages must not stop acquisition or reset accumulated totals.
6. Sensor and communication protocols are versioned and designed for backward-compatible extension.
