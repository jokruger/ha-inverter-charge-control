# Home Assistant Inverter Charge Control

Dynamic inverter grid charge current adjustment automation for Home Assistant, designed to handle weak/unstable grid conditions by intelligently adjusting charge rates based on real-time voltage monitoring.

## Overview

This automation dynamically adjusts the inverter's maximum grid charging current based on grid voltage conditions. It enables safe charging from weak grids by reducing charge current when voltage drops, and maximizing charge rate when grid conditions improve.

## Goals

- **Maintain grid connection** during weak/unstable grid conditions
- **Maximize charging efficiency** when grid is stable
- **Protect inverter EEPROM** by limiting write operations
- **Prevent grid overload** by estimating voltage drop under different charge rates
- **Automated operation** requiring minimal manual intervention

## Problem Statement

In areas with weak or unstable grid infrastructure, high charge currents can cause voltage drops that trigger inverter protection mechanisms, disconnecting the grid when charging is most needed. Manual adjustment is impractical, and conservative fixed settings waste charging opportunities when grid conditions are good.

## Solution

The automation continuously monitors grid voltage (current, minimum, and average over 15 minutes) and dynamically adjusts the battery charge current between 5A and 30A based on:

1. **Current grid state** - Whether grid is available or unavailable
2. **Voltage measurements** - Real-time voltage, 15-minute minimum, and 15-minute average
3. **Power consumption** - Current grid power draw (house load + charging)
4. **Projected voltage impact** - Estimated voltage drop if charge current is increased
5. **Battery state** - Current charge level to avoid unnecessary EEPROM writes when nearly full

### Key Logic

The automation estimates future voltage conditions using:
- **Current state estimation**: Uses actual grid power (house load + charging) to understand current voltage drop
- **Future state projection**: Estimates voltage drop for proposed charge current using:
  ```
  House load (estimated 500W) + (Charge current × Battery voltage 48V)
  ```
- **Voltage drop model**: Approximately **5V drop per 1kW** of power draw
- **Safety margin**: Uses the minimum of current voltage and 15-minute minimum voltage to ensure conservative decisions

The automation will only increase charge current if the projected voltage remains above **195V** threshold.

## Requirements

### Hardware
- **Inverter**: Deye SUN-6K-SG03LP1-EU (logic should work with other inverters with adjustable parameters)
- **Battery system**: 48V battery bank
- **Grid connection**: Single phase (currently monitors L1)

### Software
- **Home Assistant**: Version 2026.1.1 or later
- **Integration**: Solarman Stick Logger for reading/writing inverter parameters

### Required Home Assistant Entities

The following entities must be available from your inverter integration:

| Entity ID | Description | Type |
|-----------|-------------|------|
| `binary_sensor.inverter_grid` | Grid availability status | Binary Sensor |
| `sensor.inverter_grid_l1_voltage` | Real-time grid voltage (L1) | Sensor (V) |
| `sensor.inverter_grid_l1_power` | Real-time grid power draw | Sensor (W) |
| `sensor.inverter_battery` | Battery state of charge | Sensor (%) |
| `number.inverter_battery_grid_charging_current` | Grid charge current setting | Number (A) |

> **Note**: Adjust entity IDs in the configuration files to match your actual integration's entity names.

## Configuration Details

### Statistics Sensors (configuration.yaml)

Two statistics sensors track voltage behavior over 15 minutes:

```yaml
- platform: statistics
  name: inverter_grid_l1_voltage_stats_min_15m
  entity_id: sensor.inverter_grid_l1_voltage
  state_characteristic: value_min
  max_age:
    minutes: 15

- platform: statistics
  name: inverter_grid_l1_voltage_stats_avg_15m
  entity_id: sensor.inverter_grid_l1_voltage
  state_characteristic: mean
  max_age:
    minutes: 15
```

These provide:
- **Minimum voltage** over last 15 minutes (worst case scenario)
- **Average voltage** over last 15 minutes (typical conditions)

### Write Budget Counter

Protects inverter EEPROM by limiting write frequency:

```yaml
counter:
  grid_charge_writes_30m:
    name: Grid charge writes (30m)
    initial: 0
    step: 1
    minimum: 0
    maximum: 100
```

**Budget limit**: 6 writes per 30 minutes (configurable via `write_budget_30m` variable)

## Automations

### 1. Grid State Notification
Sends mobile notification when grid connects or disconnects, including voltage and battery level.

### 2. Grid Charge Write Budget Reset
Resets the write counter every 30 minutes to maintain the write budget window.

### 3. Grid Charge Write Budget Notification
Alerts when write budget is exceeded (more than 3 writes in 30 minutes).

### 4. Battery Grid Charge Current Adjustment (Main Automation)

**Triggers**:
- Grid state change (on/off)
- Voltage drops below 190V
- Time pattern (every 5 minutes)

**Decision Logic**:

| Condition | Action | Charge Current |
|-----------|--------|----------------|
| Grid off OR voltage ≤190V | **Emergency**: Set to minimum | 5A |
| Battery ≥95% | No change | Current |
| Projected voltage @ 30A ≥195V | Increase to maximum | 30A |
| Projected voltage @ 20A ≥195V | Increase to medium-high | 20A |
| Projected voltage @ 10A ≥195V | Increase to medium | 10A |
| None of above | No change | Current |

**EEPROM Protection**:
- **Minimum write interval**: 900 seconds (15 minutes) between changes
- **Write budget**: Maximum 6 writes per 30 minutes
- **Change detection**: Only writes when value actually changes
- **Budget tracking**: All writes are counted and budget enforced

> **EEPROM Life**: Deye inverters have a minimum 1 million write cycle limit. In worst case scenario this automation may write up to 8 times per hour, resulting in over 11 years of operation before reaching this limit. However, the actual write frequency is typically much lower due to stable periods (when grid is off, when grid is on and stable, or when battery is charged).

## Inverter Settings

The following inverter settings have been tested and work well with this automation:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Grid Reconnect Time | 10 minutes | Allow grid to stabilize before reconnecting |
| V Low (Normal) | 185V | Low voltage protection threshold |
| LV1 | 187V | Level 1 low voltage protection |

These settings provide appropriate protection while allowing the automation to manage charging dynamically.

## Tuning Parameters

Key variables in the main automation that can be tuned for your setup:

```yaml
variables:
  write_budget_30m: 6              # Maximum writes per 30 minutes
  min_write_interval_secs: 900     # Minimum seconds between writes
  house_load_est: 500              # Estimated house load in watts
  dv_per_kw: 5                     # Voltage drop per kW (5V/kW)
```

## Contributing

Contributions are welcome! If you improve this automation or adapt it for other inverter types, please share your changes.

## License

See [LICENSE](LICENSE) file for details.

## Acknowledgments

- Home Assistant community for the excellent home automation platform
- Solarman integration developers for enabling inverter communication
- Users dealing with weak grid infrastructure who inspired this solution
