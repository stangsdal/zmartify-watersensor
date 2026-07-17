# Flowmeter Calibration

## 1. Purpose

Each flow meter has an independent calibration factor expressed as pulses per liter. Calibration converts the authoritative pulse count into accumulated volume and current flow.

## 2. Configuration field

```text
pulses_per_liter
```

Requirements:

- stored per channel;
- greater than zero for enabled channels;
- configurable without recompiling firmware;
- persisted in NVS;
- included in diagnostic/configuration output;
- changed atomically.

## 3. Basic calculation

```text
volume_liters = pulse_count / pulses_per_liter
```

For an interval:

```text
delta_liters = delta_pulses / pulses_per_liter
flow_l_min = delta_liters * 60 / delta_time_seconds
```

## 4. Initial value

Use the manufacturer's pulses-per-liter value only as an initial estimate. Installation geometry, pressure, sensor tolerance and actual operating range may affect accuracy.

## 5. Field calibration procedure

1. Ensure the channel input is electrically stable and no false pulses are present.
2. Reset a temporary calibration-session counter; do not reset the permanent total.
3. Pass a measured reference volume through the meter.
4. Use a volume large enough to reduce reading uncertainty, preferably 20 liters or more where practical.
5. Record the exact reference volume and counted pulses.
6. Calculate:

```text
pulses_per_liter = measured_pulses / reference_liters
```

7. Apply the factor to the channel.
8. Repeat the measurement at low, normal and high expected flow.
9. Store the accepted factor and calibration metadata.

## 6. Correction from an existing factor

If the system reports `reported_liters` while the reference measurement is `actual_liters`:

```text
new_factor = old_factor * reported_liters / actual_liters
```

Example:

```text
old_factor      = 450 pulses/L
reported_liters = 21.0 L
actual_liters   = 20.0 L
new_factor      = 472.5 pulses/L
```

## 7. Calibration metadata

Store or publish:

- factor;
- calibration date/time where a clock is available;
- reference volume;
- pulse count used;
- optional operator or procedure identifier;
- firmware version;
- optional notes retained by the Edge server.

The embedded device does not need to store long free-text notes.

## 8. Validation limits

Firmware shall reject:

- zero or negative factors;
- NaN or infinite values;
- values outside a configurable plausible range;
- channel numbers outside 1–4.

A rejected update must not change the active calibration.

## 9. Low-flow validation

At low flow, compare both accumulated volume and time between pulses. A meter may have a minimum starting flow below which it does not generate reliable pulses.

Document per installed meter:

- minimum reliable flow;
- expected maximum flow;
- nominal pulses per liter;
- observed pulse width;
- installed channel;
- pipe/application description.

## 10. Acceptance criteria

Define project-specific accuracy targets. A reasonable initial engineering target may be:

- accumulated-volume error within ±2% across the normal operating range;
- no false counts during zero-flow observation;
- repeatable result across at least three reference runs;
- no lost pulses at maximum expected flow.

These values are design targets, not statements about the certification or legal metrology of the installation.
