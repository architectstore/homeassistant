# Home Assistant - SolarFlow Energy Management

This repository contains the Home Assistant configuration files for managing and optimizing a solar energy system, likely a Zendure SolarFlow or a similar battery storage setup. The automations are designed to intelligently control battery charging and discharging based on the time of day, solar production, and weather forecasts to maximize self-consumption and minimize grid dependency.

## Core Features

- **Dynamic Day/Night Modes:** Switches the system between different operational modes for daytime (prioritizing self-consumption) and nighttime (using stored energy or grid charging).
- **Grid-Zero Operation:** During the day, the system aims to match the household's power consumption with solar production, preventing energy from being exported to or imported from the grid.
- **Weather-Aware Charging:** Intelligently decides whether to charge the battery from the grid at night based on the next day's weather forecast. This ensures the battery is full before a cloudy day.
- **Scheduled Night Charging:** Provides a standard automation to charge the battery during off-peak hours at night.
- **Configurable Helpers:** Uses `input_helpers` to allow for easy adjustment of settings without editing the automations directly.

## File Structure

The main configuration is broken down into several files for better organization:

-   `config/configuration.yaml`: The main entry point for the Home Assistant configuration. It includes the other files from this repository.
-   `config/sensors.yaml`: Defines custom template sensors required for the automations to monitor power flow, battery state, and other relevant metrics.
-   `config/helpers.yaml`: Contains `input_number`, `input_boolean`, and other helpers used to control the automation logic (e.g., setting charge limits, enabling/disabling modes).
-   `config/automations/`: This directory houses the core automation logic.
    -   `10_solarflow_night-idle.yaml`: Sets the system to an idle or minimal discharge state during the night.
    -   `11_solarflow_day-grid-zero.yaml`: Manages solar power during the day to achieve zero grid interaction.
    -   `12_solarflow_night-charge-weather.yaml`: An automation that charges the battery from the grid at night, but only if the next day's weather forecast is poor.
    -   `13_solarflow_night-charge.yaml`: A standard automation for charging the battery from the grid at night, typically during cheap off-peak hours.
    -   `19_solarflow_debug.yaml`: Contains automations and tools for debugging the system and logging important values.

## Prerequisites

-   A running Home Assistant instance.
-   A solar power and battery storage system (e.g., Zendure SolarFlow) integrated into Home Assistant.
-   A weather forecast integration that provides data for the weather-aware charging automation.
-   All entity IDs for your solar inverter, battery, and power sensors must be correctly identified.

## Installation

1.  Clone this repository to your local machine.
2.  Copy the contents of the `config` directory into your Home Assistant's `/config` directory. If you already have existing `configuration.yaml`, `sensors.yaml`, etc., you will need to merge the contents of these files manually.
3.  Carefully review all files and update the entity IDs to match the ones from your own Home Assistant setup. Pay close attention to sensors, input helpers, and device names.
4.  Adjust the values in `helpers.yaml` to suit your needs (e.g., charge limits, time schedules).
5.  Go to **Developer Tools > YAML Configuration** and click **Reload Automations** and **Reload Template Entities**, or restart Home Assistant.

---

