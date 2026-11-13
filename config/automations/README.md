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
| `11_solarflow_day-grid-zero.yaml` | **Active** | Core grid-zero controller. Runs 07:00–23:00; discharges on import, charges on export. Reserves export headroom for the EV before charging the battery. |
| `12_solarflow_night-charge-weather.yaml` | **Active** | Low-tariff night grid-charge on a low-PV day: idle until 00:30, then charges each battery below its forecast-based target until 08:00. Power is headroom-capped to keep import ≤ 4100 W and ramps gradually toward that target. |
| `13_solarflow_emergency-charge.yaml` | **Active** | Manual override: while `input_boolean.solarflow_emergency_charge` is on, charges both batteries to 100 % with the same headroom/ramp logic as 12 (4100 W grid ceiling). Automations 11 and 12 are blocked while it is on. Toggle auto-switches off once all batteries are full, or after a 6 h timeout (with a notification). |
| `14_solarflow_health.yaml` | **Active** | Battery health management: sets `soc_charge_max` (90 % normally, 100 % once every 15 days). |
| `15_solarflow_grid-breaker-guard.yaml` | **Active** | Emergency safety: cuts SolarFlow AC input if grid import sustains above 4400 W for 1 minute. |
| `16_solarflow_grid-sensor-failsafe.yaml` | **Active** | Failsafe: if the Shelly reading is unavailable for 30 s (MQTT sensor has `expire_after: 60`), zeroes all SolarFlow charge/discharge limits and helpers (idle) and notifies. Re-checks every minute; 11/12/13 resume on their own when the reading returns. |
| `20_ev-charge.yaml` | **Active** | Master enable-off stop, plus night-schedule EV charging 22:00–08:00 (13 A) gated on grid headroom. |
| `21_ev-charge_sun.yaml` | **Active** | Solar-surplus EV charging (10 A): starts on sustained export, stops on sustained import (daytime only). |
| `30_lights-backyard.yaml` | Active | Backyard light on sun schedule (sunset+30 / sunrise‑60), resyncs on HA start. |
| `40_zigbee-status.yaml` | Active | ZHA sensor offline/online/daily-reminder and battery-low notifications. |
| `41_gate-sentinel.yaml` | Active | Arms/disarms Blink alarm with `input_boolean.gate_sentinel`; alerts on gate/motion and on every arm/disarm; auto-arms 5 min after both tracked phones leave Wi-Fi, auto-disarms when either returns; re-arm watchdog. |
| `50_ac-attic.yaml` | Active | Attic AC auto-cool at 26°C when attic temp > 30°C. |

The remainder of this document covers the SolarFlow/EV energy automations
(`11`–`21`) in detail.

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

Automation 11 treats each device separately: a device without a readable SoC
and limits is skipped (never assumed to be at 50 %) and the other takes the
full load. Automations 12 and 13 likewise only charge a battery that is
available. Device 2 calls use `continue_on_error: true` so a temporary offline
device does not block control.

---

## Grid Safety Architecture

Two-layer protection for the utility contract, keyed off
`sensor.shelly_phase_a_power_smoothed` (10 s moving average of the Shelly
reading; negative = export, positive = import):

| Layer | Where | Threshold | Behaviour |
|---|---|---|---|
| Proactive (headroom) | Auto 12 variables | **4100 W** | Dynamically caps charge power to `min(2000, last_charge + 4100 − current_import)`. Ramps up +500 W per ~1-min tick and down −1000 W per tick (plus a fast re-evaluation when import stays above 4100 W for 5 s). Charge below 100 W is not started. Kept below the SolarFlow's 4400 W breaker guard and the EV's 4900 W × 15 min stop threshold. |
| Reactive (breaker guard) | Auto 15 | **4400 W × 1 min** | Cuts both devices' `input_limit` to 0. Does not touch discharge. Auto 12 resumes on its next ~1‑min cycle if headroom allows. |
| Sensor loss (failsafe) | Auto 16 | Shelly unavailable **30 s** (MQTT sensor `expire_after: 60`) | Zeroes all SolarFlow charge/discharge limits and helpers (idle) and notifies; rechecks every minute. 11/12/13 resume once the reading returns. |
| EV-specific | Auto 20 (via `templates/13_grid_safety.yaml`) | start < **2000 W**, stop > **4900 W** × 15 min | Independent of the SolarFlow guards above — gates whether the EV charger is allowed to run at night. The long 15-min sustain is deliberate: short excursions up to ~5 kVA are tolerated, and it gives auto 12 plenty of time to back off first. It watches total grid import, so it also catches loads that are not the battery. |

---

## Detailed Logic

### 11 — Grid-Zero Controller (active)

**Window**: 07:00 → 23:00. The automation's own `condition` block gates
execution to this window. In addition, during the low-tariff hours
(22:00–08:00) it stands down whenever `binary_sensor.solarflow_low_pv_day`
is on, because auto 12 owns those hours then.

**Control law (rewritten)**: one signed battery command `p` (+ discharge,
− charge) is moved each cycle by `gain × (grid − target_import)` from the
previous command (damped integrator), using the **raw** `sensor.shelly_phase_a_power`
(not the smoothed one — the filter's 10 s lag was feeding stale error back
into the loop and causing the oscillation).

- **Triggers**: every raw Shelly update, a 10 s heartbeat, time edges, HA start.
  No `for:` delays.
- **Settle gate**: acts only ≥ `settle_s` (6 s) after the previous command, so
  the inverter's response is visible before correcting again.
- **Deadband**: |grid − 20 W| < 40 W → hold. `gain` 0.8, `max_step` 800 W.
- **Direction changes pass through 0** for one cycle; export while discharging
  only reduces the discharge, it never flips straight into charging.
- **SoC floors**: per battery, at `solarflow_soc_discharge_min`; a floored
  battery's share goes to the other.
- **Evening hold (from 22:00)**: a battery under 40 % SoC is not discharged
  when tomorrow's PV forecast is under 12 kWh (treated like the floor).
- **Offline / full batteries**: a device without a readable SoC or limits is
  skipped (never assumed to be at 50 %); a battery at `soc_charge_max` stops
  charging while the other continues.
- **Charge**: only when sun elevation > 3° and SoC < `soc_charge_max`,
  capped by the EV reservation.
- **Self-repair**: if device limits differ from the commanded values by > 30 W
  the limits are re-written. A new command must also differ from the last one
  by ≥ `min_delta` (30 W), unless it is 0.
- Tuning knobs are the first variables in the action block.

**EV export priority**: while `binary_sensor.ev_charger_connectivity` and
`input_boolean.ev_charge_enable` are both on and `switch.ev_charger` is off
(the EV is a candidate to start solar-charging but hasn't yet — see auto 21),
the battery's charge target is capped at `raw_surplus - ev_reserve_w` once
raw pre-battery surplus (`last_charge - grid`) exceeds 2300 W, leaving that
headroom visible as export so the EV can reach its own -2000 W start
threshold (the 300 W margin keeps the grid clearly below it) instead of the battery claiming it first. Once the EV actually
starts drawing, the cap lifts and the battery is free to use any leftover
surplus.

**Night**: this automation only runs 07:00–23:00. From 23:00 auto 12 takes
over (see below), so the limits are always governed.

---

### 12 — Low-tariff night charge (active)

**Purpose**: on a low-PV day, fill each battery during the low tariff
(22:00–08:00) to a forecast-based target so the high-tariff day is served by
the batteries.

**Shared signals** (`templates/15_solarflow_plan.yaml`):
- `sensor.solarflow_night_target_soc` = floor + (daily consumption − 60 % of the
  PV forecast) / 7.68 kWh, clamped to floor..`soc_charge_max`. Forecast = today's
  total before 12:00, tomorrow's after.
- `binary_sensor.solarflow_low_pv_day` = target > floor + 10 %.
- `input_number.solarflow_daily_consumption_kwh` (default 5; **set ≈12 for
  Dec–Feb heating season**).

**Ownership with auto 11**: 12 owns 23:00–07:00 always, and 22:00–23:00 /
07:00–08:00 only when `low_pv_day` is on (11 stands down in those hours).
Otherwise 11 owns 07:00–23:00. On a low-PV day the batteries are therefore
not discharged during cheap hours.

**Behaviour**: idle until 00:30 (limits zeroed once); then, until 08:00,
each battery below target (start hysteresis 8 %, latched by
`solarflow_night_charge_planned`) is charged,
split per battery, max 2000 W total / 1000 W per device. Charge power =
`last_charge + (4100 − grid)` capped, ramp +500 / −1000 W per tick (the grid
reading already contains the charge, so headroom is added to it). At the
07:00 / 08:00 hand-over, limits and helpers are zeroed.

The old helpers `solarflow_pv_threshold_kwh`, `solarflow_low_pv_soc_limit` and
`solarflow_evening_soc_min` (plus `solarflow_min_delta`, `solarflow_deadband`)
were removed because nothing used them.

---

### 14 — Battery Health (active)

Sets `solarflow_soc_charge_max` (it does not touch the discharge floor):
- Full cycle interval (`full_interval_days = 15`): allows 100% charge every 15
  days (the date is recorded when SoC ≥ 98.9% for 20 min), otherwise caps at 90%

Recomputed hourly and at 00:05 daily in addition to the full-cycle trigger.
Timestamp stored in `input_text.zendure_last_full_date`. Note this automation reads
`sensor.zendure_battery_soc` (single-device template sensor), not the
two-device average used elsewhere.

---

### 15 — Grid Breaker Guard (active)

**Trigger**: `sensor.shelly_phase_a_power_smoothed > 4400 W` sustained for
**1 minute** (ignores spikes).

**Action**: Sets `input_limit = 0` on both devices and zeroes
`solarflow_charge_last`. Does **not** touch `output_limit` — discharge
continues to help reduce import.

**Recovery**: Auto 12 re-evaluates on its next ~1-min cycle. The headroom
check prevents immediate re-overload.

---

### EV Automations

Both are gated by the master `input_boolean.ev_charge_enable` for starting
and routine stopping, and both target `switch.ev_charger` /
`number.ev_charger_charging_current`. The overload emergency stop (grid > 4900 W
for 15 min) ignores the toggle.

#### 20 — Scheduled Night Charging (`20_ev-charge.yaml`)

This file has **two** automations:

1. `ev_charge_stop_on_enable_off` — if `ev_charge_enable` is switched off while
   the charger is on, stops it immediately and notifies.
2. `ev_charger_night_schedule` — the actual night schedule:
   - **Start** (22:00–08:00): checks at 22:00 and when a relevant grid or
     charger state changes, including when `ev_charge_enable` turns on. It requires `ev_charge_enable = on`,
     `binary_sensor.grid_low_safe_for_ev = on` for 5 s (grid draw < 2000 W,
     from `templates/13_grid_safety.yaml`), charger currently off, EV
     connected, no charger fault. Sets current to 13 A, waits 5 s, then turns
     the charger on.
   - **Emergency stop** (any time of day, ignores the enable toggle): a
     dedicated trigger fires when `binary_sensor.grid_high_alert_ev` (grid
     draw > 4900 W) has been on for 15 minutes; if the charger is on, it is
     turned off.
   - **Scheduled stop**: turns the charger off at 08:00 (if still on and
     enabled), **unless** the session was started by auto 21 on solar
     (`input_boolean.ev_session_solar` on); auto 21 then stops it on import.
     The flag is cleared at 22:00 (the session becomes a normal night
     session), on any stop, and when a night session starts.

#### 21 — Solar Surplus Charging (`21_ev-charge_sun.yaml`)

- **Start**: a template trigger that is true while grid export ≥ 2000 W, EV
  connected, enable on and charger off, held for 1 minute → sets current to
  10 A (the night schedule uses 13 A), waits 5 s, turns the charger on. Because
  it is a template trigger, plugging in or enabling the EV while export is
  already high also starts the timer.
- **Stop**: grid import ≥ 500 W sustained for 5 minutes, **or** battery discharge
  command (`input_number.solarflow_discharge_last`, excludes PV pass-through) > 1200 W
  sustained for 15 minutes (the session is running on stored energy; the -2000 W start
  threshold is below the EV's ~2300 W draw, so the battery may bridge the
  ramp-up briefly) → turns the charger off, **but only while inside 08:00–22:00**. Outside that window, stopping
  the charger on import is left to auto 20's grid-safety emergency stop.

---

## Key Helpers

| Helper | Default | Purpose |
|---|---|---|
| `input_number.solarflow_discharge_last` | 0 (max 1600) | Total discharge limit last set (both devices) |
| `input_number.solarflow_charge_last` | 0 (max 2000) | Total charge limit last set (both devices) |
| `input_number.solarflow_soc_discharge_min` | 25 | Configurable minimum SoC floor for discharge, set directly in the helper |
| `input_number.solarflow_soc_charge_max` | 100 | Max SoC for solar charge absorption (managed by auto 14) |
| `input_number.solarflow_daily_consumption_kwh` | 5 | Expected high-tariff-hours consumption, used by `sensor.solarflow_night_target_soc` (set ≈12 for Dec–Feb) |
| `input_boolean.solarflow_night_charge_planned` | off | Latch flag used only by auto 12 (keeps a started night charge from aborting on a restart) |
| `input_boolean.ev_charge_enable` | — | Master EV charging kill switch, read by autos 20 and 21 |
| `input_boolean.ev_session_solar` | off | Set by auto 21 when it starts a solar session; makes auto 20's 08:00 stop skip it |

---

## Common Pitfalls

- **Check traces**: Developer Tools → Traces → `solarflow_day_grid_zero_step_100w`
  for IDLE / branch-taken reasons.
- **Auto 11 stops at 23:00**: it does not idle the batteries at night itself —
  see the note under "11 — Grid-Zero Controller" above. Auto 12's
  23:00–00:30 branch picks up the handoff immediately (hard-edge trigger at
  23:00), so there should be no ungoverned gap in normal operation.
- **EV + SolarFlow overlap (daytime, export)**: automation 11 reserves 2300 W
  of export for the EV before charging the battery (see "EV export priority"
  above), so the battery no longer races the EV for solar surplus.
- **EV + SolarFlow overlap (nighttime, import)**: if both EV and SolarFlow
  charge simultaneously at night, the headroom logic in auto 12 backs off the
  SolarFlow charge when import approaches 4100 W (fast 5 s trigger plus ramp).
  A brief excursion above the 4.6 kVA contract is tolerated; the breaker does
  not trip immediately. Auto 15 is the final backstop
  at 4400 W for SolarFlow; auto 20's own 4900 W/15 min alert is the backstop
  for the EV charger.
- **Shelly sign convention**: negative = export, positive = import. If
  inverted, reverse all thresholds in autos 11, 12, 15, 20, and 21.
- **Device 2 offline**: autos 11, 12 and 13 skip the unavailable device and run
  on the other one alone. Service calls use `continue_on_error: true` so the
  controllers keep running.
- **`planned` flag timing**: auto 12 re-evaluates `planned` every minute
  (plus on SoC changes and at 22:00, 23:00, 00:30, 07:00, 08:00 and HA start),
  so it stays fresh right up to when the acting window opens at 00:30.
- **10 and 13 were removed (2026-10-11)**: both were long-disabled legacy
  night-idle/night-charge automations, superseded by auto 11's own window and
  auto 12's handoff-idle branch. If you see references to them in old traces
  or notes, they no longer exist.
