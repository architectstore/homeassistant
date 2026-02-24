# Automation Logic

This directory contains the core logic for the SolarFlow energy management system. Files are numbered by operational lifecycle: day grid-zero, night charge, etc.

## 📂 File Overview

| File | Type | Description |
|------|------|-------------|
| `10_solarflow_night-idle.yaml` | **State Management** | Night idle mode post-sunset. |
| `11_solarflow_day-grid-zero.yaml` | **Core Logic** | **07:00–01:00**: 100W-step grid-zero (discharge on import>75W, charge daytime only). SoC lock ≤50% after 22:00. [file:4] |
| `12_solarflow_night-charge-weather.yaml` | **Predictive** | **01:00–07:00**: 1000W grid charge if PV forecast<5kWh (SoC<70%) OR emergency (SoC≤20%→45%). [file:2] |
| `13_solarflow_night-charge.yaml` | **Scheduled** | Off-peak mandatory charge. |
| `20_ev-charge.yaml` | **EV Integration** | **22:00–08:00**: Off-peak scheduled EV charging (safety: connectivity check, fault prevention). |
| `21_ev-charge_sun.yaml` | **EV Solar Surplus** | Solar-surplus EV charging: Start on export ≤-2500W, stop on import ≥500W (5s hold). 10A initial current. |
| `19_solarflow_debug.yaml` | **Maintenance** | Health monitoring & logs. |

## ⚙️ Detailed Logic

### 🌙 Night Operations (01:00–07:00 → No overlap)

**`11_solarflow_day-grid-zero.yaml`** extends to **00:00–01:00** (discharge if load>75W, SoC>min).[file:4]

#### `12_solarflow_night-charge-weather.yaml` (**Smart Charge, 01:00–07:00**)
- **Triggers**: Every 10min, SoC change, 01:00/07:00.
- **Low PV**: `sensor.energy_production_today < input_number.solarflow_pv_threshold_kwh` (5kWh) **AND** SoC<70 **AND** (SoC<45 **OR** already charging).
- **Emergency**: SoC≤`solarflow_emergency_soc`(20%) → charge until ≥`solarflow_emergency_stop_soc`(45%).
- **Power**: Fixed 1000W input.[file:2]

#### `13_solarflow_night-charge.yaml` (**Hard Charge**)
Off-peak base SoC fill (30%+).[file:2]

### ☀️ Day Operations (07:00–01:00)

#### `11_solarflow_day-grid-zero.yaml` (**Core Grid-Zero**)
- **Window**: 07:00–01:00 cross-midnight (`t >=07:00 OR t<01:00`).[file:4]
- **Triggers**: Import>75W/5s, export<-75W/5s, **every 1min**, 07/08/22:00, HA start.
- **Discharge** (`output_limit>0`): If grid>120W **AND** SoC>`soc_discharge_min`(25) → step to match load-50W (50W steps, max 800W).[file:4]
- **Charge** (`input_limit>0`): Daytime only (sunrise+30min→sunset-30min), export logic (2-1000W steps).[file:4]
- **22:00 Lock**: If SoC≤50 → force idle (both limits=0).[file:4]
- **01:00 Hard Stop**: Forces idle regardless.[file:4]
- **Mode**: `restart` (last trigger wins).[file:4]

**Why intermittent discharge?** Stops on SoC≤min(25), 22:00 lock(≤50), low load(≤75W), or `min_delta=40` hysteresis → restarts on load spike + SoC recovery.[file:4]

### 🛠️ Utilities

**`19_solarflow_debug.yaml`**: MQTT/offline alerts, mode-switch logs.

**`20_ev-charge.yaml`**: 22:00–08:00 scheduled EV charging (safety: connectivity, no faults).

**`21_ev-charge_sun.yaml`**: Solar-surplus EV charging during daytime.

## 🔗 Dependencies

- **Sensors** (`../sensors.yaml`): `sensor.shelly_phase_a_power_smoothed`, `sensor.solarflow_800_pro_electric_level`, `sensor.energy_production_today`.
- **Helpers** (`../helpers.yaml`): `input_number.solarflow_*` (soc_discharge_min=25, pv_threshold_kwh=5, etc.), `input_number.solarflow_charge_last/discharge_last`.
- **EV** (`tuya.local`): `switch.ev_charger`, `sensor.ev_charger_*`.

### 🚗 EV Charging Operations

#### `20_ev-charge.yaml` (**Scheduled Night Charging**)
- **Window**: 22:00–08:00 (off-peak hours).
- **Triggers**: Time-based (22:00 to enable, 08:00 to disable).
- **Safety Checks**:
  - `binary_sensor.ev_charger_connectivity == 'on'` (EV must be plugged in).
  - `binary_sensor.ev_charger_problem == 'off'` (no charger faults).
  - Only toggles if current state differs from desired state.
- **Actions**: Turns charger on/off via `switch.ev_charger`.
- **Notifications**: Creates persistent notification with status, power, and connectivity info.

#### `21_ev-charge_sun.yaml` (**Solar Surplus Charging**)
- **Mode**: `restart` (latest trigger wins to prevent oscillation).
- **Condition**: `binary_sensor.ev_charger_connectivity == 'on'` (EV must be connected).
- **Start Logic**:
  - **Trigger**: `sensor.shelly_phase_a_power_smoothed ≤ -2500W` (grid export).
  - **Pre-start**: Sets `number.ev_charger_charging_current = 10A`.
  - **Delay**: 2-second wait to ensure current limit is applied.
  - **Action**: Turns on `switch.ev_charger`.
- **Stop Logic**:
  - **Trigger**: `sensor.shelly_phase_a_power_smoothed ≥ 500W` for 5 seconds (grid import).
  - **Action**: Turns off `switch.ev_charger`.
- **Key Parameters**:
  - Export threshold: `-2500W` (adjustable for your solar capacity).
  - Import threshold: `500W` (with 5s hold to prevent flapping).
  - Initial current: `10A` (safe starting point; adjust based on available surplus).

**Solar Surplus Charging Logic:**
- **Negative Shelly values = Export to grid** (excess solar production).
- **Positive Shelly values = Import from grid** (consuming more than producing).
- Starts charging only when exporting significant power (≥2.5kW).
- Stops charging if importing power (to avoid grid consumption).
- Uses smoothed power sensor to reduce noise and prevent rapid cycling.

## ⚠️ Common Pitfalls

- **Overlap fixed**: Grid-zero ends 01:00 → night charge starts 01:00.
- **Check traces**: Developer Tools → Traces → `solarflow_day_grid_zero_step_100w` for "IDLE"/"LOCKED" reasons.
- **Tune helpers**: Adjust `soc_discharge_min`, `evening_soc_min=50`, `pv_threshold_kwh` via UI.[file:4]
- **EV Sensor Sign Convention**: Verify your Shelly reports **negative values for export** and **positive for import**. If inverted, reverse the threshold logic in `21_ev-charge_sun.yaml`.
- **EV Current Steps**: Ensure `number.ev_charger_charging_current` supports the configured value (10A). Adjust if your EVSE uses different steps.
- **Anti-Flapping**: The 5-second hold on import threshold prevents rapid on/off cycles. Increase if experiencing oscillation.
- **Smoothed Sensor**: Use a time-averaged power sensor (e.g., 5-second mean) to reduce noise from instantaneous power fluctuations.
