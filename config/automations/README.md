# SolarFlow Automation Logic

This directory contains the core logic for the SolarFlow energy management system. Files are numbered by operational lifecycle: day grid-zero, night charge, etc.

## 📂 File Overview

| File | Type | Description |
|------|------|-------------|
| `10_solarflow_night-idle.yaml` | **State Management** | Night idle mode post-sunset. |
| `11_solarflow_day-grid-zero.yaml` | **Core Logic** | **07:00–01:00**: 100W-step grid-zero (discharge on import>75W, charge daytime only). SoC lock ≤50% after 22:00. [file:4] |
| `12_solarflow_night-charge-weather.yaml` | **Predictive** | **01:00–07:00**: 1000W grid charge if PV forecast<5kWh (SoC<70%) OR emergency (SoC≤20%→45%). [file:2] |
| `13_solarflow_night-charge.yaml` | **Scheduled** | Off-peak mandatory charge. |
| `20_ev-charge.yaml` | **EV Integration** | Tuya EV off-peak scheduling. |
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

**`20_ev-charge.yaml`**: 22:00–08:00 Tuya EV (safety: connectivity, no faults).

## 🔗 Dependencies

- **Sensors** (`../sensors.yaml`): `sensor.shelly_phase_a_power_smoothed`, `sensor.solarflow_800_pro_electric_level`, `sensor.energy_production_today`.
- **Helpers** (`../helpers.yaml`): `input_number.solarflow_*` (soc_discharge_min=25, pv_threshold_kwh=5, etc.), `input_number.solarflow_charge_last/discharge_last`.
- **EV** (`tuya.local`): `switch.ev_charger`, `sensor.ev_charger_*`.

## ⚠️ Common Pitfalls

- **Overlap fixed**: Grid-zero ends 01:00 → night charge starts 01:00.
- **Check traces**: Developer Tools → Traces → `solarflow_day_grid_zero_step_100w` for "IDLE"/"LOCKED" reasons.
- **Tune helpers**: Adjust `soc_discharge_min`, `evening_soc_min=50`, `pv_threshold_kwh` via UI.[file:4]
