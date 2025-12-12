# SolarFlow Automation Logic

This directory contains the core logic for the SolarFlow energy management system. The files are numbered to organize the operational lifecycle of the system, covering day/night cycles, grid-zero optimization, and predictive charging.

## 📂 File Overview

| File | Type | Description |
| :--- | :--- | :--- |
| `10_solarflow_night-idle.yaml` | **State Management** | Handles the system state during non-charging night hours. |
| `11_solarflow_day-grid-zero.yaml` | **Core Logic** | Dynamic output control to match home consumption (Grid Zero). |
| `12_solarflow_night-charge-weather.yaml` | **Predictive** | Conditional grid charging based on tomorrow's solar forecast. |
| `13_solarflow_night-charge.yaml` | **Scheduled** | Standard off-peak grid charging routines. |
| `19_solarflow_debug.yaml` | **Maintenance** | Logging and debugging tools for system troubleshooting. |

## ⚙️ Detailed Logic

### 🌙 Night Operations

#### `10_solarflow_night-idle.yaml`
*   **Trigger:** Sunset or Solar power < 10W.
*   **Purpose:** Transitions the system into "Idle" mode when solar production ends.
*   **Behavior:**
    *   Disables high-frequency polling used during the day.
    *   Sets inverter output to 0W (or min_load) to prevent battery drain when energy is not needed.
    *   Acts as a safety stop to ensure the battery doesn't discharge below critical levels overnight outside of specific demand windows.

#### `12_solarflow_night-charge-weather.yaml` (Smart Charge)
*   **Trigger:** Specific time (e.g., 23:00) or Tariff change.
*   **Purpose:** Optimizes battery levels based on *future* production.
*   **Behavior:**
    *   Checks the `sensor.energy_production_forecast_tomorrow`.
    *   **Logic:** If `forecast < threshold`, initiate grid charging to `target_soc`.
    *   Prevents buying grid energy if tomorrow is sunny enough to fill the battery naturally.

#### `13_solarflow_night-charge.yaml` (Hard Charge)
*   **Trigger:** Low tariff window start.
*   **Purpose:** Mandatory base-load charging.
*   **Behavior:**
    *   Forces battery charging from the grid regardless of weather.
    *   Typically used to ensure a minimum SoC (e.g., 30%) for the morning peak before the sun rises.
    *   Controlled by `input_boolean.night_charge_enabled` in `../helpers.yaml`.

### ☀️ Day Operations

#### `11_solarflow_day-grid-zero.yaml`
*   **Trigger:** High frequency (e.g., every 10-30 seconds) or Power Sensor state change.
*   **Purpose:** Minimizes grid import/export.
*   **Behavior:**
    *   Calculates `Home Demand - Solar Production`.
    *   Adjusts the SolarFlow output hub to match the deficit.
    *   **Priority:** Prioritizes powering the home first, then directs excess energy to the battery.

### 🛠️ Utilities

#### `19_solarflow_debug.yaml`
*   **Purpose:** System health and debugging.
*   **Behavior:**
    *   Sends notifications if the battery disconnects or MQTT goes offline.
    *   Logs specific variable states when switching between Day/Night modes to assist with tuning automation thresholds.

## 🔗 Dependencies

These automations rely on entities defined in the parent directory:
*   **Sensors (`../sensors.yaml`):** Requires real-time power consumption and battery SoC sensors.
*   **Helpers (`../helpers.yaml`):** Uses input booleans to manually override automation logic (e.g., `maintenance_mode`).
