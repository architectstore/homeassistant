# Home Assistant - Energy Management

This repository contains Home Assistant configuration files for managing and optimizing a solar energy system (e.g., Zendure SolarFlow) using Shelly devices for power monitoring. The automations are designed to intelligently control battery charging/discharging **and EV charging** based on time of day and real-time grid power flow to maximize self-consumption and minimize grid dependency.

## Core Features

- **Dynamic Day/Night Modes:** Switches the system between different operational modes for daytime (prioritizing self-consumption) and nighttime (using stored energy or grid charging).
- **Grid-Zero Operation:** During the day, the system aims to match household power consumption with solar production, preventing energy from being exported to or imported from the grid.
- **Weather-Aware Charging:** Decides whether to charge the battery from the grid at night based on the next day's weather forecast.
- **Scheduled Night Charging:** Provides a standard automation to charge the battery during off-peak hours at night.
- **Configurable Helpers:** Uses helpers (`input_number`, `input_boolean`, etc.) for easy adjustment of settings without editing the automations directly.

### EV Charging Capabilities

#### Scheduled Night Charging (`20_ev-charge.yaml`)
- **Off-peak scheduling:** Automatically enables EV charging at 22:00 and disables at 08:00.
- **Safety checks:** Only operates when EV is connected (`binary_sensor.ev_charger_connectivity: on`) and no charger faults are present (`binary_sensor.ev_charger_problem: off`).
- **Notifications:** Provides persistent notifications with charger status, power consumption, and connectivity information.

#### Solar Surplus Charging (`21_ev-charge_sun.yaml`)
- **Solar-surplus enablement:** EV charging is only allowed when the EVSE reports it is connected (`binary_sensor.ev_charger_connectivity: on`).
- **Export-based start logic (Shelly negative = export):** When grid power indicates export beyond a configurable threshold (≤ -2500W on smoothed Shelly power sensor), the automation prepares and starts charging.
- **Pre-start current setpoint:** Sets the charger current via `number.ev_charger_charging_current` to a safe initial value (10A default) before enabling the charger.
- **Controlled start sequencing:** Applies current limit first, waits 2 seconds, then turns on the charger switch (`switch.ev_charger`) to avoid race conditions with EVSEs that read the limit at start.
- **Import-based stop logic:** If grid power transitions to consumption/import beyond threshold (≥ 500W for 5 seconds), charging stops by turning off `switch.ev_charger` to avoid pulling from the grid.
- **Anti-flapping protection:** Uses smoothed 5-second averaged Shelly sensor and 5-second hold time on import threshold to prevent rapid toggling.
- **Restart mode:** Latest trigger wins, ensuring quick response to changing conditions while preventing oscillation.

## File Structure

The main configuration is broken down into several files for better organization:

- `config/configuration.yaml`: Main entry point for Home Assistant configuration; includes other files and directories.
- `config/helpers.yaml`: Contains helpers used to control automation logic (e.g., charge limits, enabling/disabling modes, thresholds).
- `config/sensors/`: Custom template and integration sensors.
  - `10_zendure-rest.yaml`: RESTful sensors to pull data from the Zendure SolarFlow API (battery SoC, output power, etc.).
  - `11_shelly.yaml`: Shelly sensors (e.g., Shelly Pro/3EM) for real-time monitoring of grid and home power flow (including smoothed power used for EV surplus control).
- `config/automations/`: Core automation logic for battery management and EV charging. This directory contains a separate `README.md` with detailed documentation for each automation:
  - `10_solarflow_night-idle.yaml`: Night idle mode management
  - `11_solarflow_day-grid-zero.yaml`: Daytime grid-zero operation (07:00-01:00)
  - `12_solarflow_night-charge-weather.yaml`: Weather-aware night charging (01:00-07:00)
  - `13_solarflow_night-charge.yaml`: Scheduled night charging
  - `19_solarflow_debug.yaml`: System health monitoring and debugging
  - `20_ev-charge.yaml`: Scheduled off-peak EV charging (22:00-08:00)
  - `21_ev-charge_sun.yaml`: Solar surplus EV charging (dynamic)

## Prerequisites

- A running Home Assistant instance.
- A Zendure SolarFlow battery storage system integrated via a RESTful API.
- A Shelly device (e.g., Shelly Pro 3EM) for accurate, real-time power monitoring of grid flow and consumption.
- A weather forecast integration providing data for the weather-aware charging automation (e.g., `sensor.energy_production_today`).
- An EV charger/EVSE integrated into Home Assistant (e.g., via Tuya) exposing:
  - `switch.ev_charger` (charging enable/disable)
  - `binary_sensor.ev_charger_connectivity` (on when EV is plugged in)
  - `binary_sensor.ev_charger_problem` (fault detection)
  - `number.ev_charger_charging_current` (adjustable amperage, typically 6-16A in 2A steps)
  - `sensor.ev_charger_status` and `sensor.ev_charger_power` (monitoring)

## Installation

1. Clone this repository to your local machine.
2. Copy the contents of the `config` directory into your Home Assistant `/config` directory (merge manually if you have existing files).
3. Ensure your `configuration.yaml` includes files from `sensors` and `automations` directories, for example:
   ```
   automation: !include_dir_merge_list automations/
   sensor: !include_dir_merge_list sensors/
   ```
4. Review all files and update entity IDs, IP addresses, and API keys to match your setup:
   - Zendure API endpoints and authentication
   - Shelly device IP addresses and entity IDs
   - EV charger entity IDs (adapt to your specific EVSE integration)
   - Weather forecast sensor names
5. Adjust values in `helpers.yaml` to suit your needs:
   - Battery charge/discharge limits and thresholds
   - SoC minimums for different times of day
   - PV forecast threshold for weather-aware charging
   - EV charging thresholds (export/import levels for solar surplus mode)
6. **Important for EV Solar Surplus Charging:** Verify your Shelly power sensor reports **negative values for export** and **positive values for import**. If your sensor uses opposite sign convention, adjust the thresholds in `21_ev-charge_sun.yaml` accordingly.
7. Go to **Developer Tools > YAML Configuration** and click **Reload Automations** and **Reload Template Entities**, or restart Home Assistant.

## Documentation

For detailed information about each automation's logic, timing, triggers, and troubleshooting tips, see the comprehensive documentation in `config/automations/README.md`.

## Configuration Tips

- **Battery Management:** Start with conservative SoC limits (e.g., `soc_discharge_min: 25`) and adjust based on your usage patterns and battery health.
- **EV Solar Surplus:** The default export threshold (-2500W) works for systems with 3-4kW of solar panels. Adjust based on your typical export levels and EV charging requirements.
- **Anti-Flapping:** If experiencing rapid on/off cycling, increase the smoothing period on power sensors or add longer hold times to triggers.
- **Weather Forecast:** Ensure your weather integration provides accurate daily PV production forecasts for optimal battery charging decisions.

## Monitoring and Debugging

- Use **Developer Tools > Traces** to inspect automation execution and understand why certain actions were taken or skipped.
- Check the persistent notifications created by the scheduled EV charger for status updates.
- Monitor the helper values (`input_number.*`) to see real-time adjustments made by the automations.
- Enable debug logging for specific automations using the `19_solarflow_debug.yaml` automation.