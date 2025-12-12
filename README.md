# Home Assistant - SolarFlow Energy Management

This repository contains the Home Assistant configuration files for managing and optimizing a solar energy system, likely a Zendure SolarFlow, using Shelly devices for power monitoring. The automations are designed to intelligently control battery charging and discharging based on time of day, solar production, and weather forecasts to maximize self-consumption and minimize grid dependency.

## Core Features

- **Dynamic Day/Night Modes:** Switches the system between different operational modes for daytime (prioritizing self-consumption) and nighttime (using stored energy or grid charging).
- **Grid-Zero Operation:** During the day, the system aims to match the household's power consumption with solar production, preventing energy from being exported to or imported from the grid.
- **Weather-Aware Charging:** Intelligently decides whether to charge the battery from the grid at night based on the next day's weather forecast.
- **Scheduled Night Charging:** Provides a standard automation to charge the battery during off-peak hours at night.
- **Configurable Helpers:** Uses `input_helpers` to allow for easy adjustment of settings without editing the automations directly.

## File Structure

The main configuration is broken down into several files for better organization:

-   `config/configuration.yaml`: The main entry point for the Home Assistant configuration. It includes the other configuration files and directories.
-   `config/helpers.yaml`: Contains `input_number`, `input_boolean`, and other helpers used to control the automation logic (e.g., setting charge limits, enabling/disabling modes).
-   `config/sensors/`: This directory defines all custom template and integration sensors.
    -   `10_zendure-rest.yaml`: Defines RESTful sensors to pull data directly from the Zendure SolarFlow API (e.g., battery SoC, output power).
    -   `11_shelly.yaml`: Defines sensors for Shelly devices, likely a Shelly Pro/3EM used for real-time grid and home consumption monitoring.
-   `config/automations/`: This directory houses the core automation logic. It contains a separate `README.md` file that details the function of each automation.

## Prerequisites

-   A running Home Assistant instance.
-   A Zendure SolarFlow battery storage system integrated via a RESTful API.
-   A Shelly device (e.g., Shelly Pro 3EM) for accurate, real-time power monitoring of your home and the grid.
-   A weather forecast integration that provides data for the weather-aware charging automation.

## Installation

1.  Clone this repository to your local machine.
2.  Copy the contents of the `config` directory into your Home Assistant's `/config` directory. If you already have existing files, you will need to merge the contents manually.
3.  Ensure your `configuration.yaml` is set up to include files from the `sensors` and `automations` directories. For example:
    ```
    automation: !include_dir_merge_list automations/
    sensor: !include_dir_merge_list sensors/
    ```
4.  Carefully review all files and update the entity IDs, IP addresses, and API keys to match your own setup.
5.  Adjust the values in `helpers.yaml` to suit your needs (e.g., charge limits, time schedules).
6.  Go to **Developer Tools > YAML Configuration** and click **Reload Automations** and **Reload Template Entities**, or restart Home Assistant.

---

