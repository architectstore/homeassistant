# Automation Logic

This directory contains the core logic for the SolarFlow energy management system.
Files are numbered by operational lifecycle. Two Zendure SolarFlow 800 Pro devices are managed as a fleet: all power limits are tracked in **total watts** and split 50/50 per device on write.

---

## File Overview

| File | Status | Description |
|------|--------|-------------|
| `10_solarflow_night-idle.yaml` | **DISABLED** | Legacy night idle enforcer. Replaced by auto 11's built-in idle logic. |
| `11_solarflow_day-grid-zero.yaml` | **Active** | Core grid-zero controller. Discharge 07:00–22:00, idle 22:00–08:00 when charge is planned or SoC is low. |
| `12_solarflow_night-charge-weather.yaml` | **Active** | Dynamic night charge 22:00–08:00 from grid when PV forecast is low. Power is headroom-capped to keep import ≤ 4500 W. |
| `13_solarflow_night-charge.yaml` | **DISABLED** | Unconditional SoC-based night charge. Headroom guard (4500 W) is already in place if re-enabled. |
| `14_solaflow_health.yaml` | **Active** | Battery health management: adjusts `soc_charge_max` and `soc_discharge_min` based on full/deep cycle intervals. |
| `15_solarflow_grid-breaker-guard.yaml` | **Active** | Emergency safety: cuts all AC input if grid import sustains above 5000 W for 1 minute. |
| `19_solarflow_debug.yaml` | **Active** | MQTT/offline alerts and mode-switch logs. |
| `20_ev-charge.yaml` | **Active** | Scheduled EV charging 22:00–08:00 (off-peak). |
| `21_ev-charge_sun.yaml` | **Active** | Solar-surplus EV charging: starts on export ≤ -2500 W, stops on import ≥ 500 W. |

---

## Device Fleet

Both devices are always commanded together. Helpers store **total watts**; each device receives `value / 2`.

| Entity pattern | Device 1 | Device 2 |
|---|---|---|
| Output limit | `number.solarflow_800_pro_output_limit` | `number.solarflow_800_pro_2_output_limit` |
| Input limit | `number.solarflow_800_pro_input_limit` | `number.solarflow_800_pro_2_input_limit` |
| AC mode | `select.solarflow_800_pro_ac_mode` | `select.solarflow_800_pro_2_ac_mode` |
| SoC | `sensor.solarflow_800_pro_electric_level` | `sensor.solarflow_800_pro_2_electric_level` |

SoC used in all automations is the **average** of both devices (`float(50)` default if either is offline). Device 2 calls use `continue_on_error: true` so a temporary offline device does not block control.

---

## Grid Safety Architecture

Two-layer protection for the **single-phase 5000 W utility contract**:

| Layer | Where | Threshold | Behaviour |
|---|---|---|---|
| Proactive (headroom) | Auto 12 & 13 variables | **4500 W** | Dynamically caps charge power to `min(2000, 4500 − current_import)`. Skips cycle if headroom < 200 W. |
| Reactive (breaker guard) | Auto 15 | **5000 W × 1 min** | Cuts both device `input_limit` to 0. Does not touch discharge. Auto 12 resumes on next 30-min cycle if headroom allows. |

---

## Detailed Logic

### 11 — Grid-Zero Controller (active)

**Window**: 07:00 → 22:00 (core grid-zero period). Can start at 07:00 if `night_charge_planned = OFF`, otherwise waits until 08:00.

**Triggers**: Grid > 75 W for 5 s, grid < -100 W for 10 s, every 1 min, time edges, HA start.

**Tuning parameters (total across both devices)**:
- `step_size`: 100 W (50 W per device)
- `max_discharge`: 1600 W
- `max_charge`: 2000 W (solar export absorption only)
- `min_delta`: 40 W (hysteresis before applying a new limit)

**Decision flow (in priority order)**:

1. **Night idle rule** — if time is in 22:00–08:00 window:
   - `planned = ON` → **always idle** (auto 12 owns this window; prevents wasteful discharge before refill)
   - `planned = OFF` AND SoC ≤ `evening_soc_min` (45%) → **idle** (preserve battery reserve)
   - `planned = OFF` AND SoC > `evening_soc_min` → continues grid-zero discharge down to `soc_discharge_min`

2. **Branch A — Discharge**: grid > 120 W AND SoC > `soc_discharge_min` → stepped output up to 1600 W total. Helpers track total; each device gets `final_discharge / 2`.

3. **Branch B — Solar charge**: grid < -80 W AND sun above horizon (sunrise+30 min → sunset-30 min) → stepped input up to 2000 W total. **Dynamic window — follows actual sunrise/sunset, not fixed times.**

4. **Default — Idle**: both targets zero and state needs resetting.

**Hard stop at 01:59:30**: fires only when `planned = ON`; forces all limits to 0 before auto 12's charge window (now redundant since auto 12 starts at 22:00, but kept as a safety fence for the pre-02:00 period).

---

### 12 — Night Charge Weather (active)

**Purpose**: Charge both batteries from the grid during low-tariff hours when tomorrow's PV forecast is low.

**Planning phase** (runs any time, every 30 min + SoC change):
- Reads `sensor.solar_production_estimate_today_total` (sum of all PV forecast sensors)
- If forecast < `solarflow_pv_threshold_kwh` (5 kWh) AND avg SoC < 70% AND (SoC < `low_pv_soc_limit` OR already charging) → sets `input_boolean.solarflow_night_charge_planned = ON`
- This flag is also read by auto 11 to trigger its idle rule

**Charge window**: **22:00–08:00** (full low-tariff period)

**Dynamic power (evaluated every cycle)**:
```
grid_headroom    = max(0, 4500 − current_grid_import)
safe_charge_power = min(2000, grid_headroom)
per_device_limit  = safe_charge_power / 2
```
- Skips charging if `safe_charge_power < 200 W`
- Accounts for EV charging automatically (EV reduces headroom → SolarFlow backs off)
- Stores `safe_charge_power` (total) in `solarflow_charge_last` helper so auto 11 grid-zero math stays consistent

**Window end (08:00)**: clears `solarflow_night_charge_planned = OFF`.

---

### 13 — Night Charge Unconditional (DISABLED)

Charges at up to 2000 W total when SoC < 50%, stops at 90%. Same 4500 W headroom guard as auto 12 is already implemented. Re-enable if you want an unconditional SoC-floor charge separate from the weather-based logic.

---

### 14 — Battery Health (active)

Adjusts `solarflow_soc_charge_max` and `solarflow_soc_discharge_min` based on:
- Full cycle interval (`full_interval_days = 15`): allows 100% charge every 15 days, otherwise caps at 90%
- Deep cycle interval (`discharge_interval_days = 20`): allows 5% discharge floor every 20 days, otherwise 20%

Timestamps stored in `input_text.zendure_last_full_date` and `input_text.zendure_last_deep_discharge_date`.

---

### 15 — Grid Breaker Guard (active)

**Trigger**: `sensor.shelly_phase_a_power_smoothed > 5000 W` sustained for **1 minute** (ignores spikes).

**Action**: Sets `input_limit = 0` on both devices. Does **not** touch `output_limit` — discharge continues to help reduce import.

**Recovery**: Auto 12 re-evaluates on its next 30-min cycle. The headroom check prevents immediate re-overload.

---

### EV Automations

#### 20 — Scheduled Night Charging
- **Window**: 22:00–08:00
- Requires `binary_sensor.ev_charger_connectivity = on` and no faults
- Turns `switch.ev_charger` on/off; sends persistent notification

#### 21 — Solar Surplus Charging
- **Start**: grid export ≤ -2500 W → sets current to 10 A, starts charger
- **Stop**: grid import ≥ 500 W for 5 s → stops charger
- Uses smoothed Shelly sensor to prevent rapid cycling

---

## Key Helpers

| Helper | Default | Purpose |
|---|---|---|
| `input_number.solarflow_discharge_last` | 0 (max 1600) | Total discharge limit last set (both devices) |
| `input_number.solarflow_charge_last` | 0 (max 2000) | Total charge limit last set (both devices) |
| `input_number.solarflow_soc_discharge_min` | 20 | Absolute SoC floor for discharge |
| `input_number.solarflow_evening_soc_min` | 45 | SoC floor for post-22:00 idle (when planned=OFF) |
| `input_number.solarflow_soc_charge_max` | 90 | Max SoC for solar charge absorption |
| `input_number.solarflow_pv_threshold_kwh` | 5 | PV forecast threshold to trigger night charge |
| `input_number.solarflow_low_pv_soc_limit` | 40 | SoC limit for the low-PV charge decision |
| `input_boolean.solarflow_night_charge_planned` | off | Shared flag: auto 12 sets it, auto 11 reads it |

---

## Common Pitfalls

- **Check traces**: Developer Tools → Traces → `solarflow_day_grid_zero_step_100w` for NIGHT-IDLE / LOCKED reasons.
- **EV + SolarFlow overlap**: If both EV and SolarFlow charge simultaneously at night, headroom logic in auto 12 backs off SolarFlow charge. Auto 15 is the final backstop at 5000 W.
- **Shelly sign convention**: Negative = export, positive = import. If inverted, reverse all thresholds in auto 11 and 21.
- **Device 2 offline**: SoC defaults to 50% average; service calls use `continue_on_error: true` so auto 11/12 keep running on device 1 alone.
- **planned flag timing**: Auto 12 sets `planned` throughout the day. Auto 11 respects it from 22:00 onwards. If forecast changes close to 22:00, auto 11 reacts on its next 1-min cycle.
