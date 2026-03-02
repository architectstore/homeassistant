# Home Assistant - Energy Management

This repository contains Home Assistant configuration files for managing and optimizing a solar energy system (e.g., Zendure SolarFlow) using Shelly devices for power monitoring. The automations are designed to intelligently control battery charging/discharging **and EV charging** based on time of day and real-time grid power flow to maximize self-consumption and minimize grid dependency. [file:36]

## Core Features

- **Dynamic Day/Night Modes:** Switches the system between different operational modes for daytime (prioritizing self-consumption) and nighttime (using stored energy or grid charging). [file:36]
- **Grid-Zero Operation:** During the day, the system aims to match household power consumption with solar production, preventing energy from being exported to or imported from the grid. [file:36]
- **Weather-Aware Charging:** Decides whether to charge the battery from the grid at night based on the next day's weather forecast. [file:36]
- **Scheduled Night Charging:** Provides a standard automation to charge the battery during off-peak hours at night. [file:36]
- **Battery Health Management:** Manages both the *upper* SoC limit (daily cap, allow 100% every N calendar days) **and** the *lower* SoC limit (normal discharge floor, allow deeper discharge down to a configured value if not reached for N calendar days) by adjusting helpers used by the main battery controller automations. [file:36][file:1]
- **Configurable Helpers:** Uses helpers (`input_number`, `input_boolean`, `input_text`, etc.) for easy adjustment of settings without editing the automations directly. [file:36]

### EV Charging Capabilities

#### Scheduled Night Charging (`20_ev-charge.yaml`)
- **Off-peak scheduling:** Automatically enables EV charging at 22:00 and disables at 08:00. [file:36]
- **Safety checks:** Only operates when EV is connected (`binary_sensor.ev_charger_connectivity: on`) and no charger faults are present (`binary_sensor.ev_charger_problem: off`). [file:36]
- **Notifications:** Provides persistent notifications with charger status, power consumption, and connectivity information. [file:36]

#### Solar Surplus Charging (`21_ev-charge_sun.yaml`)
- **Solar-surplus enablement:** EV charging is only allowed when the EVSE reports it is connected (`binary_sensor.ev_charger_connectivity: on`). [file:36]
- **Export-based start logic (Shelly negative = export):** When grid power indicates export beyond a configurable threshold (≤ -2500W on smoothed Shelly power sensor), the automation prepares and starts charging. [file:36]
- **Pre-start current setpoint:** Sets the charger current via `number.ev_charger_charging_current` to a safe initial value (10A default) before enabling the charger. [file:36]
- **Controlled start sequencing:** Applies current limit first, waits 2 seconds, then turns on the charger switch (`switch.ev_charger`) to avoid race conditions with EVSEs that read the limit at start. [file:36]
- **Import-based stop logic:** If grid power transitions to consumption/import beyond threshold (≥ 500W for 5 seconds), charging stops by turning off `switch.ev_charger` to avoid pulling from the grid. [file:36]
- **Anti-flapping protection:** Uses smoothed 5-second averaged Shelly sensor and 5-second hold time on import threshold to prevent rapid toggling. [file:36]
- **Restart mode:** Latest trigger wins, ensuring quick response to changing conditions while preventing oscillation. [file:36]

## File Structure

The main configuration is broken down into several files for better organization: [file:36]

- `config/configuration.yaml`: Main entry point for Home Assistant configuration; includes other files and directories. [file:36]
- `config/helpers.yaml`: Contains helpers used to control automation logic (e.g., charge limits, enabling/disabling modes, thresholds). [file:36]
- `config/sensors/`: Custom template and integration sensors. [file:36]
  - `10_zendure-rest.yaml`: RESTful sensors to pull data from the Zendure SolarFlow API (battery SoC, output power, etc.). [file:36]
  - `11_shelly.yaml`: Shelly sensors (e.g., Shelly Pro/3EM) for real-time monitoring of grid and home power flow (including smoothed power used for EV surplus control). [file:36]
- `config/automations/`: Core automation logic for battery management and EV charging. This directory contains a separate `README.md` with detailed documentation for each automation: [file:36]
  - `10_solarflow_night-idle.yaml`: Night idle mode management [file:36]
  - `11_solarflow_day-grid-zero.yaml`: Daytime grid-zero operation (07:00-01:00) [file:36]
  - `12_solarflow_night-charge-weather.yaml`: Weather-aware night charging (01:00-07:00) [file:36]
  - `13_solarflow_night-charge.yaml`: Scheduled night charging [file:36]
  - `14_solarflow_health.yaml`: Battery health management (upper SoC cap and lower SoC discharge floor, both with “exercise” intervals) [file:36]
  - `19_solarflow_debug.yaml`: System health monitoring and debugging [file:36]
  - `20_ev-charge.yaml`: Scheduled off-peak EV charging (22:00-08:00) [file:36]
  - `21_ev-charge_sun.yaml`: Solar surplus EV charging (dynamic) [file:36]

## Prerequisites

- A running Home Assistant instance. [file:36]
- A Zendure SolarFlow battery storage system integrated via a RESTful API. [file:36]
- A Shelly device (e.g., Shelly Pro 3EM) for accurate, real-time power monitoring of grid flow and consumption. [file:36]
- A weather forecast integration providing data for the weather-aware charging automation (e.g., `sensor.energy_production_today`). [file:36]
- An EV charger/EVSE integrated into Home Assistant (e.g., via Tuya) exposing: [file:36]
  - `switch.ev_charger` (charging enable/disable) [file:36]
  - `binary_sensor.ev_charger_connectivity` (on when EV is plugged in) [file:36]
  - `binary_sensor.ev_charger_problem` (fault detection) [file:36]
  - `number.ev_charger_charging_current` (adjustable amperage, typically 6-16A in 2A steps) [file:36]
  - `sensor.ev_charger_status` and `sensor.ev_charger_power` (monitoring) [file:36]

### Battery health prerequisites

The battery health automation (`14_solarflow_health.yaml`) expects: [file:36]

- `sensor.zendure_battery_soc`: Battery SoC sensor used to detect “full” events and “deep discharge” events. [file:36]
- `input_number.solarflow_soc_charge_max`: Helper that the day grid-zero controller uses as the maximum allowed SoC for charging. [file:36][file:1]
- `input_number.solarflow_soc_discharge_min`: Helper that the day grid-zero controller uses as the minimum allowed SoC for discharging (discharge floor). [file:1]
- `input_text.zendure_last_full_date`: Helper that stores the last “full charge” date as `YYYY-MM-DD`. [file:36]
- `input_text.zendure_last_deep_discharge_date`: Helper that stores the last “deep discharge” date as `YYYY-MM-DD`. [file:1]


