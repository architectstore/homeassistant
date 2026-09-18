# Home Assistant Configuration

Personal Home Assistant configuration for the house. Covers solar/battery energy
management (dual Zendure SolarFlow 800 Pro batteries, EcoFlow microinverters, and
a Shelly Pro 3EM for grid metering), EV charging, and a set of smaller home
security/comfort automations (garage, gate sentinel/Blink, Zigbee device health,
attic AC, backyard lights).

## Repository Layout

- `config/configuration.yaml` — main entry point. Loads `automations/`,
  `sensors/`, and `templates/` directories, the three `helpers/*.yaml` files,
  and declares one inline MQTT sensor for the raw Shelly grid power reading.
- `config/automations/` — all automations (auto-loaded as a list). See
  [config/automations/README.md](config/automations/README.md) for detailed
  logic on the SolarFlow/EV energy automations.
- `config/sensors/` — REST, integration (energy), and filter sensors.
- `config/templates/` — template `sensor`/`binary_sensor` definitions (derived
  values, aggregations, safety flags).
- `config/helpers/` — `input_number`, `input_boolean` (file is named
  `input_bolean.yaml`), and `input_text` helper definitions used by the
  automations.

`configuration.yaml` also references `scripts.yaml`, `scenes.yaml`, and a
`themes/` directory; these are managed through the Home Assistant UI and are
not present in this repo.

## Energy Management (SolarFlow)

Two Zendure SolarFlow 800 Pro units are controlled as a single fleet: every
power limit is computed in **total watts** and split 50/50 across both
devices when written. Grid reference is `sensor.shelly_phase_a_power`, smoothed
with a 10‑second moving average into `sensor.shelly_phase_a_power_smoothed`
(`config/sensors/11_shelly.yaml`) — this smoothed sensor is what every control
loop and safety guard actually reads.

Full per-automation logic, tuning values, and the grid-safety architecture are
documented in [config/automations/README.md](config/automations/README.md).
Summary of the active pieces:

| File | Status | Purpose |
|---|---|---|
| `11_solarflow_day-grid-zero.yaml` | Active | Core grid-zero controller, 07:00–22:00: discharges on import, charges from solar export. |
| `12_solarflow_night-charge-weather.yaml` | Active | Plans a night grid-charge anytime; actually charges 02:00–07:00 when tomorrow's PV forecast is low, headroom-capped. |
| `14_solaflow_health.yaml` | Active | Raises/lowers the SoC charge cap and discharge floor based on how long since the battery last reached 100%/5%. |
| `15_solarflow_grid-breaker-guard.yaml` | Active | Emergency stop: cuts AC charge on both devices if import sustains > 4400 W for 1 minute. |
| `19_solarflow_debug.yaml` | Active | Logs every discharge/charge limit change for troubleshooting. |
| `10_solarflow_night-idle_disabled.yaml` | Disabled | Legacy unconditional night idle (23:00–08:00). |
| `13_solarflow_night-charge_disabled.yaml` | Disabled | Legacy unconditional SoC-based night charge. |

## EV Charging

Both EV automations respect a master `input_boolean.ev_charge_enable` toggle;
turning it off immediately stops an in-progress charge
(`20_ev-charge.yaml`, first automation in the file).

- **`20_ev-charge.yaml` — night schedule**: between 22:00 and 08:00, starts
  charging (10 A) only while `binary_sensor.grid_low_safe_for_ev` (grid draw
  < 2000 W, from `config/templates/13_grid_safety.yaml`) has been on for 5 s
  and the EV is connected with no charger fault. It stops immediately if
  `binary_sensor.grid_high_alert_ev` (grid draw > 4300 W) holds for 2 minutes,
  or unconditionally at 08:00.
- **`21_ev-charge_sun.yaml` — solar-surplus charging**: starts when export
  reaches ≥ 2500 W for 1 minute; stops when import reaches ≥ 500 W for 5
  minutes, but only while inside the 08:00–22:00 window (outside it, the
  night-schedule automation above owns stop/start behaviour).

Both rely on an EVSE integration exposing `switch.ev_charger`,
`binary_sensor.ev_charger_connectivity`, `binary_sensor.ev_charger_problem`,
and `number.ev_charger_charging_current`.

## Home Security & Monitoring

- **`00_garage_trigger.yaml`** — notifies if the garage gate sensor is `on`
  (open) while no motion has been detected in the garage for 30 minutes.
- **`01_garage_water.yaml`** — notifies immediately on a garage water-leak
  sensor trigger.
- **`41_gate-sentinel.yaml`** — `input_boolean.gate_sentinel` arms/disarms a
  Blink alarm panel and, while armed, pushes a notification on gate opening or
  entrance/garage motion. Auto-arms 5 minutes after **both**
  `device_tracker.honor_2` and `device_tracker.sonia_s_a56_2` (phones, via
  AsusRouter) leave the home Wi-Fi, and auto-disarms as soon as **either**
  reconnects. Every arm/disarm (manual or automatic) sends a push
  notification. A watchdog re-arms Blink every 30 minutes if it was disarmed
  externally while the sentinel is still on.
- **`40_zigbee-status.yaml`** — notifies when any of a fixed list of ZHA
  Zigbee sensors goes unavailable/comes back, sends a daily 09:00 reminder for
  sensors still offline, and alerts when tracked device batteries drop below
  20%.

## Climate & Lighting

- **`50_ac-attic.yaml`** — sets the attic AC to cool at 26°C when the attic
  room temperature exceeds 30°C; notifies on both auto-on and manual-off.
- **`30_lights-backyard.yaml`** — turns a backyard switch on 30 minutes after
  sunset and off 1 hour before sunrise, resyncing on Home Assistant startup.

## Sensors & Templates

- **`sensors/10_zendure-rest.yaml`** + **`templates/10_zendure.yaml`** — polls
  the Zendure SolarFlow REST API (`properties`/`packData`) every 15 s and
  derives per-device power/SoC/status sensors, per-pack SoC, and fleet totals
  (`sensor.solarflow_total_output_home_power`, etc.); plus energy (kWh)
  integration sensors for battery charge/discharge and solar production.
- **`sensors/11_shelly.yaml`** + the inline MQTT sensor in
  `configuration.yaml` — raw Shelly Pro 3EM phase‑A power over MQTT, smoothed
  into `sensor.shelly_phase_a_power_smoothed` (10 s moving average). This is
  the primary signal for every SolarFlow and EV automation.
- **`sensors/12_ecoflow_total.yaml`** + **`templates/12_ecoflow.yaml`** —
  sums two microinverter AC power sensors and integrates to an energy sensor.
- **`templates/11_solar.yaml`** — aggregates four `energy_production_today*`
  forecast sensors into `sensor.solar_production_estimate_today_total`, and
  combines Solax/microinverter/SolarFlow instantaneous power into
  `sensor.pv_power_total_aggregated`.
- **`templates/13_grid_safety.yaml`** — `binary_sensor.grid_low_safe_for_ev`
  (< 2000 W) and `binary_sensor.grid_high_alert_ev` (> 4300 W), gating the EV
  night-schedule automation.
- **`templates/14_solarflow-pv.yaml`** — sums the 4 solar-string power
  sensors on each of the two SolarFlow devices into
  `sensor.solarflow_pv_power_total`.

## Helpers

- `helpers/input_number.yaml` — `solarflow_discharge_last` /
  `solarflow_charge_last` (last commanded total watts), `solarflow_soc_discharge_min`
  / `solarflow_soc_charge_max` (SoC floor/cap, managed by auto 14),
  `solarflow_evening_soc_min`, `solarflow_low_pv_soc_limit`,
  `solarflow_pv_threshold_kwh`. `solarflow_min_delta` and `solarflow_deadband`
  are defined but currently unused (auto 11 hard-codes its own `min_delta`).
- `helpers/input_bolean.yaml` — `solarflow_night_charge_planned` (set by auto
  12), `gate_sentinel`, `ev_charge_enable`.
- `helpers/input_text.yaml` — `zendure_last_full_date`,
  `zendure_last_deep_discharge_date` (read/written by auto 14).

## Prerequisites

- A running Home Assistant instance.
- Two Zendure SolarFlow 800 Pro units reachable via their local REST API
  (`http://<ip>/properties/report`).
- A Shelly Pro 3EM (or similar) publishing phase‑A power over MQTT.
- Microinverter/Solax PV sensors and 4 PV forecast sensors
  (`sensor.energy_production_today[_2/_3/_4]`) for the aggregation templates.
- An EV charger/EVSE integrated into Home Assistant (e.g. via Tuya) exposing
  `switch.ev_charger`, `binary_sensor.ev_charger_connectivity`,
  `binary_sensor.ev_charger_problem`, `number.ev_charger_charging_current`.
- A Blink alarm panel (`alarm_control_panel.blink_queijas_home`) and the
  AsusRouter integration (`device_tracker.honor_2`,
  `device_tracker.sonia_s_a56_2`) for the gate sentinel automation.
- ZHA-integrated Zigbee sensors matching the entity list in
  `40_zigbee-status.yaml`.
- The `notify.mobile_app_v5` mobile app service, used by nearly every
  notification in this repo.
