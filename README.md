[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-Donate-orange?style=for-the-badge&logo=buy-me-a-coffee)](https://buymeacoffee.com/orquitto)
[![Build and Release Firmware](https://github.com/orkanm/HASensors/actions/workflows/build_and_release.yml/badge.svg)](https://github.com/orkanm/HASensors/actions/workflows/build_and_release.yml)

# HASensors — DIY Home Assistant Air Quality Sensor

**HASensors** is a battery-powered, ultra-low-power air quality sensor built on the **ESPHome** ecosystem and powered by the **ESP32-C3**. It measures temperature, humidity, and atmospheric pressure using the **BME280** sensor and pushes readings directly to **Home Assistant** before entering deep sleep to conserve battery.

---

## 🏛️ Architecture: Ultra-Low Power Design

Unlike always-on sensors, HASensors is purpose-built around deep sleep to maximize battery life.

- **Wake → Sense → Report → Sleep**: The device wakes every 30 minutes, connects to Wi-Fi, publishes readings to HA, then immediately returns to deep sleep.
- **OTA-Aware**: A Home Assistant `input_boolean.keep_sensors_awake` helper switch keeps the device alive for firmware updates when needed.
- **Fail-Safe Hardware Timer**: A 30-second hardware `run_duration` ensures the device sleeps even if network connectivity fails entirely.
- **Zero Maintenance**: Readings are pushed directly to Home Assistant's native API with no intermediate broker required.

---

## ✨ Features

- **BME280 Sensor**: Measures temperature (°C), relative humidity (%), and atmospheric pressure (hPa)
- **Deep Sleep**: 30-minute sleep cycles with ~5-second wake windows for maximum battery life
- **OTA Updates**: Controlled via a Home Assistant helper switch — flip it ON to keep the device awake for flashing
- **Safe Mode**: 1-minute post-flash protection window (if new firmware causes a boot loop, the device stays flashable)
- **Multi-Network Wi-Fi**: Supports up to 3 SSID/password pairs
- **Fallback AP**: Captive portal hotspot if no configured network is reachable
- **Calibration Filters**: Per-sensor lambda filters to apply individual calibration offsets

---

## 🔋 Power Budget

| State | Duration |
|-------|----------|
| Deep Sleep | ~30 minutes |
| Wake (switch OFF) | ~5 seconds |
| Wake (switch ON / OTA) | Indefinite |
| Fail-safe max wake | 30 seconds |

---

## 🛠️ Hardware

### Components

| Component | Part |
|-----------|------|
| MCU | ESP32-C3 DevKitM-1 |
| Sensor | BME280 (I²C, address `0x76`) |
| Power | LiPo battery + TP4056 charger module |

### Wiring Diagram (ASCII)

```text
     ESP32-C3 DevKitM-1                  BME280 Module
  +------------------------+           +--------------+
  |   GPIO20 (PWR)  -------+-----------+-> VCC        |
  |   GPIO10 (GND)  -------+-----------+-> GND        |
  |   GPIO8  (SDA)  -------+-----------+-> SDA        |
  |   GPIO9  (SCL)  -------+-----------+-> SCL        |
  |                        |           |   SDO --> GND| (addr 0x76)
  +------------------------+           +--------------+
```

> **Note**: GPIO20 and GPIO10 are used as software-controlled power rails for the BME280.
> GPIO20 is configured `ALWAYS_ON` (outputs 3.3V) and GPIO10 is `ALWAYS_OFF` (outputs GND).
> This allows future software power-cycling of the sensor if needed.

### Pin Mapping Table

| ESP32-C3 Pin | BME280 Pin | Type | Notes |
| :--- | :--- | :--- | :--- |
| **GPIO20** | VCC | Power | Software-controlled 3.3V (`ALWAYS_ON`) |
| **GPIO10** | GND | Power | Software-controlled GND (`ALWAYS_OFF`) |
| **GPIO8** | SDA | I²C Data | ⚠️ Strapping pin — no external pull-down |
| **GPIO9** | SCL | I²C Clock | ⚠️ Strapping pin — no external pull-down |
| — | SDO → GND | Address select | Sets I²C address to `0x76` |

---

## 🚀 Quick Start

### Prerequisites

- [ESPHome](https://esphome.io) installed
- Home Assistant with the ESPHome integration
- A `secrets.yaml` file (never committed — see `.gitignore`)

### secrets.yaml format

```yaml
wifi_ssid: "YourSSID"
wifi_password: "YourPassword"
2_wifi_ssid: "SecondSSID"
2_wifi_password: "SecondPassword"
3_wifi_ssid: "ThirdSSID"
3_wifi_password: "ThirdPassword"
AirQuality_api_key: "your-encrypted-api-key"
AirQuality_ota_password: "your-ota-password"
AirQuality_ap_password: "your-fallback-ap-password"
```

### Flash

```bash
esphome run TempHumidPres.yaml
```

### OTA Wake Window

In Home Assistant, toggle `input_boolean.keep_sensors_awake` to **ON** to keep the device awake, then flash normally. Toggle it back **OFF** when done — the device will immediately go to sleep.

---

## 🏗️ CI/CD Pipeline

Every push to `main` automatically:
1. Installs ESPHome from `requirements.txt`
2. Creates a dummy `secrets.yaml` with placeholder values
3. Compiles `TempHumidPres.yaml` and verifies it builds without errors
4. Uploads the compiled `firmware.factory.bin` as a workflow artifact

Tagging a release (`v*`) additionally creates a GitHub Release with the binary attached.

---

## 📖 Home Assistant Setup

1. Add a **Helper** → `input_boolean` named `keep_sensors_awake`
2. The sensor will automatically appear in HA after first boot via the ESPHome integration
3. Entities published:
   - `sensor.air_quality_temperature`
   - `sensor.air_quality_humidity`
   - `sensor.air_quality_pressure`
   - `sensor.air_quality_deep_sleep_uptime`

---

## ☕ Support the Project

If you find this useful, consider buying me a coffee!

[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-Donate-orange?style=for-the-badge&logo=buy-me-a-coffee)](https://buymeacoffee.com/orquitto)
