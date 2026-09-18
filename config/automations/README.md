# Automation Logic

This directory contains the core logic for the SolarFlow energy management system,
EV charging, and the house's smaller security/comfort automations.
Files are numbered by operational lifecycle. Two Zendure SolarFlow 800 Pro devices
are managed as a fleet: all power limits are tracked in **total watts** and split
50/50 per device on write.

---

## File Overview

| File | Status | Description |
|------|--------|-------------|
| `00_garage_trigger.yaml` | Active | Notifies if the garage gate is open with no garage motion for 30 min. |
| `01_garage_water.yaml` | Active | Notifies immediately on a garage water-leak detection. |
| `10_solarflow_night-idle_disabled.yaml` | **DISABLED** | Legacy unconditional night idle enforcer (23:00–08:00). Replaced by auto 11's own condition gate. |
| `11_solarflow_day-grid-zero.yaml` | **Active** | Core grid-zero controller. Only runs 07:00–22:00; discharges on import, charges on export. |
| `12_solarflow_night-charge-weather.yaml` | **Active** | Plans a night grid-charge anytime; actually charges 02:00–07:00 when tomorrow's PV forecast is low. Power is headroom-capped to keep import ≤ 4500 W. |
| `13_solarflow_night-charge_disabled.yaml` | **DISABLED** | Legacy unconditional SoC-based night charge (23:00–08:00). Same 4500 W headroom guard already in place if re-enabled. |
| `14_solaflow_health.yaml` | **Active** | Battery health management: adjusts `soc_charge_max` and `soc_discharge_min` based on full/deep cycle intervals. |
| `15_solarflow_grid-breaker-guard.yaml` | **Active** | Emergency safety: cuts SolarFlow AC input if grid import sustains above 4400 W for 1 minute. |
| `19_solarflow_debug.yaml` | **Active** | Logs discharge/charge limit changes at info level. |
| `20_ev-charge.yaml` | **Active** | Master enable-off stop, plus night-schedule EV charging 22:00–08:00 gated on grid headroom. |
| `21_ev-charge_sun.yaml` | **Active** | Solar-surplus EV charging: starts on sustained export, stops on sustained import (daytime only). |
| `30_lights-backyard.yaml` | Active | Backyard light on sun schedule (sunset+30 / sunrise‑60), resyncs on HA start. |
| `40_zigbee-status.yaml` | Active | ZHA sensor offline/online/daily-reminder and battery-low notifications. |
| `41_gate-sentinel.yaml` | Active | Arms/disarms Blink alarm with `input_boolean.gate_sentinel`; alerts on gate/motion and on every arm/disarm; auto-arms 5 min after both tracked phones leave Wi-Fi, auto-disarms when either returns; re-arm watchdog. |
| `50_ac-attic.yaml` | Active | Attic AC auto-cool at 26°C when attic temp > 30°C. |

The remainder of this document covers the SolarFlow/EV energy automations
(`10`–`21`) in detail.

---

## Device Fleet

Both devices are always commanded together. Helpers store **total watts**; each
device receives `value / 2`.

| Entity pattern | Device 1 | Device 2 |
|---|---|---|
| Output limit | `number.solarflow_800_pro_output_limit` | `number.solarflow_800_pro_2_output_limit` |
| Input limit | `number.solarflow_800_pro_input_limit` | `number.solarflow_800_pro_2_input_limit` |
| AC mode | `select.solarflow_800_pro_ac_mode` | `select.solarflow_800_pro_2_ac_mode` |
| SoC | `sensor.solarflow_800_pro_electric_level` | `sensor.solarflow_800_pro_2_electric_level` |

SoC used in all automations is the **average** of both devices (`float(50)` or
`float(0)` default depending on the file if either is offline). Device 2 calls
use `continue_on_error: true` so a temporary offline device does not block
control.

---

## Grid Safety Architecture

Two-layer protection for the utility contract, keyed off
`sensor.shelly_phase_a_power_smoothed` (10 s moving average of the Shelly
reading; negative = export, positive = import):

| Layer | Where | Threshold | Behaviour |
|---|---|---|---|
| Proactive (headroom) | Auto 12 & 13 variables | **4500 W** | Dynamically caps charge power to `min(2000, 4500 − current_import)`. Skips the cycle if headroom < 200 W. |
| Reactive (breaker guard) | Auto 15 | **4400 W × 1 min** | Cuts both devices' `input_limit` to 0. Does not touch discharge. Auto 12 resumes on its next 30‑min cycle if headroom allows. |
| EV-specific | Auto 20 (via `templates/13_grid_safety.yaml`) | start < **2000 W**, stop > **4300 W** × 2 min | Independent of the SolarFlow guards above — gates whether the EV charger is allowed to run at night. |

---

## Detailed Logic

### 11 — Grid-Zero Controller (active)

**Window**: 07:00 → 22:00. The automation's own `condition` block gates
execution to this window — it does not trigger or act outside it, regardless
of the night-related variables computed inside its logic (see note below).

**Triggers**: grid > 75 W for 5 s, grid < ‑100 W for 10 s, every 1 min, time
edges (07:00, 08:00, 22:00), HA start.

**Tuning parameters (total across both devices)**:
- `step_size`: 100 W (50 W per device)
- `max_discharge`: 1600 W
- `max_charge`: 2000 W (solar export absorption only)
- `min_delta`: 40 W (hysteresis before applying a new limit)

**Decision flow (in priority order)**:

1. **Idle rule (`should_idle`)** — idle if `input_number.solarflow_charge_last`
   is already > 0 (auto 12 is charging), OR if the internal `night_charge_window`
   flag (22:00–08:00) is true AND PV forecast < 10 kWh AND SoC ≤
   `solarflow_evening_soc_min` (45%). In practice the second branch can only
   ever be true in the narrow overlap where the automation is still inside its
   07:00–22:00 condition window (i.e. right at the 22:00 edge trigger), since
   the automation does not run at all outside that window.

2. **Branch A — Discharge**: grid > 120 W AND SoC above the effective floor
   (5% during 08:00–22:00 "high tariff"; `solarflow_soc_discharge_min` — 25%
   default — otherwise) → stepped output up to 1600 W total, each device gets
   `final_discharge / 2`.

3. **Branch B — Solar charge**: grid < ‑80 W AND sun is up (sunrise+30 min →
   sunset‑30 min) → stepped input up to 2000 W total.

4. **Default — Idle**: both targets zero and state needs resetting.

**Note on night behaviour**: because the automation's `condition` restricts it
to 07:00–22:00, nothing in this file adjusts the SolarFlow limits between
22:00 and 07:00. Whatever discharge/charge value was last written before the
window closed stays in effect until auto 12 (charging window) or auto 15
(breaker guard) changes it, or auto 11 restarts the following day at 07:00.

---

### 12 — Night Charge Weather (active)

**Purpose**: Charge both batteries from the grid during low-tariff hours when
today's/tomorrow's PV forecast is low.

**Planning phase** (runs any time, every 30 min + on SoC change + at 01:50 to
refresh just before auto 11's old cutoff):
- Reads `sensor.solar_production_estimate_today_total` (sum of 4 PV forecast
  sensors, see `templates/11_solar.yaml`)
- If forecast < `solarflow_pv_threshold_kwh` (5 kWh) AND avg SoC < 70% AND
  (SoC < `low_pv_soc_limit` (40%) OR already charging) →
  `input_boolean.solarflow_night_charge_planned = ON`
- This flag is also read by auto 11's (currently unreachable — see above)
  night-idle branch.

**Charge window (acts on the inverter)**: **02:00–07:00** only. Outside that
window the automation only updates the planning flag; it explicitly `stop`s
before touching any inverter entity.

**Dynamic power (evaluated every cycle, inside 02:00–07:00)**:
```
grid_headroom     = max(0, 4500 − current_grid_import)
safe_charge_power = min(2000, grid_headroom)
per_device_limit  = safe_charge_power / 2
```
- Skips charging if `safe_charge_power < 200 W`
- Accounts for EV charging automatically (EV import reduces headroom →
  SolarFlow charge backs off)
- Stores `safe_charge_power` (total) in `solarflow_charge_last` so auto 11's
  math stays consistent
- At 07:00 the plan flag is cleared (`solarflow_night_charge_planned = OFF`)

---

### 13 — Night Charge Unconditional (DISABLED)

Charges at up to 2000 W total when SoC < 50%, stops at 90%. Active 23:00–08:00.
Same 4500 W headroom guard as auto 12 is already implemented. Re-enable if you
want an unconditional SoC-floor charge separate from the weather-based logic.

---

### 14 — Battery Health (active)

Adjusts `solarflow_soc_charge_max` and `solarflow_soc_discharge_min` based on:
- Full cycle interval (`full_interval_days = 15`): allows 100% charge every 15
  days (triggered by SoC ≥ 98.9% for 20 min), otherwise caps at 90%
- Deep cycle interval (`discharge_interval_days = 20`): allows a 5% discharge
  floor every 20 days (triggered by SoC ≤ 5.1% for 10 min), otherwise 20%

Recomputed hourly and at 00:05 daily in addition to the trigger events.
Timestamps stored in `input_text.zendure_last_full_date` and
`input_text.zendure_last_deep_discharge_date`. Note this automation reads
`sensor.zendure_battery_soc` (single-device template sensor), not the
two-device average used elsewhere.

---

### 15 — Grid Breaker Guard (active)

**Trigger**: `sensor.shelly_phase_a_power_smoothed > 4400 W` sustained for
**1 minute** (ignores spikes).

**Action**: Sets `input_limit = 0` on both devices and zeroes
`solarflow_charge_last`. Does **not** touch `output_limit` — discharge
continues to help reduce import.

**Recovery**: Auto 12 re-evaluates on its next 30-min cycle. The headroom
check prevents immediate re-overload.

---

### EV Automations

Both are gated by the master `input_boolean.ev_charge_enable`, and both
target `switch.ev_charger` / `number.ev_charger_charging_current`.

#### 20 — Scheduled Night Charging (`20_ev-charge.yaml`)

This file has **two** automations:

1. `ev_charge_stop_on_enable_off` — if `ev_charge_enable` is switched off while
   the charger is on, stops it immediately and notifies.
2. `ev_charger_night_schedule` — the actual night schedule:
   - **Start** (22:00–08:00): requires `ev_charge_enable = on`,
     `binary_sensor.grid_low_safe_for_ev = on` for 5 s (grid draw < 2000 W,
     from `templates/13_grid_safety.yaml`), charger currently off, EV
     connected, no charger fault. Sets current to 10 A, waits 5 s, then turns
     the charger on.
   - **Emergency stop**: if `binary_sensor.grid_high_alert_ev` (grid draw >
     4300 W) holds for 2 minutes while charging, turns the charger off.
   - **Scheduled stop**: unconditionally turns the charger off at 08:00 (if
     still on and enabled).

#### 21 — Solar Surplus Charging (`21_ev-charge_sun.yaml`)

- **Start**: grid export ≥ 2500 W sustained for 1 minute → sets current to
  10 A, waits 5 s, turns the charger on. Requires EV connected, enable on,
  charger currently off.
- **Stop**: grid import ≥ 500 W sustained for 5 minutes → turns the charger
  off, **but only while inside 08:00–22:00**. Outside that window, stopping
  the charger on import is left to auto 20's grid-safety emergency stop.

---

## Key Helpers

| Helper | Default | Purpose |
|---|---|---|
| `input_number.solarflow_discharge_last` | 0 (max 1600) | Total discharge limit last set (both devices) |
| `input_number.solarflow_charge_last` | 0 (max 2000) | Total charge limit last set (both devices) |
| `input_number.solarflow_soc_discharge_min` | 10 | Absolute SoC floor for discharge (managed by auto 14, default overridden by it) |
| `input_number.solarflow_evening_soc_min` | 45 | SoC floor for post-22:00 idle decision in auto 11 |
| `input_number.solarflow_soc_charge_max` | 100 | Max SoC for solar charge absorption (managed by auto 14) |
| `input_number.solarflow_pv_threshold_kwh` | 5 | PV forecast threshold to trigger night charge |
| `input_number.solarflow_low_pv_soc_limit` | 40 | SoC limit for the low-PV charge decision |
| `input_boolean.solarflow_night_charge_planned` | off | Shared flag: auto 12 sets it, auto 11 reads it |
| `input_boolean.ev_charge_enable` | — | Master EV charging kill switch, read by autos 20 and 21 |
| `input_number.solarflow_min_delta`, `input_number.solarflow_deadband` | — | Defined but currently **unused** — auto 11 hard-codes its own `min_delta` (40 W) instead |

---

## Common Pitfalls

- **Check traces**: Developer Tools → Traces → `solarflow_day_grid_zero_step_100w`
  for IDLE / branch-taken reasons.
- **Auto 11 stops at 22:00**: it does not idle the batteries at night itself —
  see the note under "11 — Grid-Zero Controller" above. A discharge limit set
  just before 22:00 can persist overnight until auto 12 or 15 touches it.
- **EV + SolarFlow overlap**: if both EV and SolarFlow charge simultaneously
  at night, the headroom logic in auto 12 backs off the SolarFlow charge.
  Auto 15 is the final backstop at 4400 W for SolarFlow; auto 20's own
  4300 W/2 min alert is the backstop for the EV charger.
- **Shelly sign convention**: negative = export, positive = import. If
  inverted, reverse all thresholds in autos 11, 12, 15, 20, and 21.
- **Device 2 offline**: SoC defaults to 50% (or 0% in auto 12) average;
  service calls use `continue_on_error: true` so the controllers keep running
  on device 1 alone.
- **`planned` flag timing**: auto 12 sets `planned` throughout the day. If the
  forecast changes close to 22:00, auto 12 reacts on its next 30-min planning
  cycle or the 01:50 pre-cutoff refresh.
