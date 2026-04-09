# Software Design Document — HASensors Firmware

**Project:** HASensors  
**Firmware file:** `TempHumidPres.yaml`  
**Platform:** ESPHome 2026.x · ESP32-C3 · esp-idf framework  
**Version:** v0.1.0  
**Last updated:** 2026-04-09  

---

## 1. Purpose and Scope

This document describes the software architecture, state machine, design decisions, and constraints for the HASensors ESPHome firmware. It targets contributors and maintainers who need to understand, modify, or extend the firmware.

HASensors is a **battery-powered, ultra-low-power air quality sensor node** that measures temperature, humidity, and atmospheric pressure (BME280) and reports the values to Home Assistant (HA) via the native ESPHome API. The primary software constraint is maximising battery life while maintaining data accuracy.

---

## 2. System Overview

```
┌──────────────────────────────────────────────────────────────┐
│                          ESP32-C3                            │
│                                                              │
│  ┌──────────────┐   I²C    ┌───────────────────────────┐     │
│  │   BME280     │◄────────►│  ESPHome BME280 Component │     │
│  │  (0x76)      │          │  update_interval: never   │     │
│  └──────────────┘          └───────────────────────────┘     │
│                                          │ component.update  │
│  GPIO20 (VCC) ──────────────────────┐    │ (manual trigger)  │
│  GPIO10 (GND) ──────────────────────┘    │                   │
│                                          ▼                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                 on_connect Automation                  │  │
│  │  wait api → prevent sleep → check ota_mode_ha          │  │
│  │  ├─ ON  → stay awake (sensor_readings_blocked=true)    │  │
│  │  └─ OFF → unblock → trigger → 2s delay → sleep         │  │
│  └────────────────────────────────────────────────────────┘  │
│                            │                                 │
│                      Wi-Fi / mDNS                            │
└──────────────────────────────────────────────────────────────┘
                             │  Native ESPHome API (encrypted)
                             ▼
                ┌────────────────────────┐
                │    Home Assistant      │
                │  input_boolean.        │
                │  keep_sensors_awake    │
                └────────────────────────┘
```

---

## 3. Component Inventory

| ESPHome Component | ID | Role |
|---|---|---|
| `deep_sleep` | `deep_sleep_1` | Controls sleep/wake cycle |
| `globals` | `sensor_readings_blocked` | Boot-safe read-block flag |
| `binary_sensor` (homeassistant) | `ota_mode_ha` | Mirrors HA helper switch |
| `switch` (gpio) | `pwr_pin_sensor` | Soft VCC for BME280 (GPIO20) |
| `switch` (gpio) | `gnd_pin_sensor` | Soft GND for BME280 (GPIO10) |
| `sensor` (bme280_i2c) | `bme280_sensor` | Temperature / Humidity / Pressure |
| `sensor` (uptime) | `uptime_sensor` | Boot cycle diagnostics |
| `wifi` | — | Multi-network, LIGHT power-save |
| `api` | — | Noise-encrypted HA connection |
| `ota` | — | OTA firmware update target |
| `safe_mode` | — | Boot-loop rollback protection |

---

## 4. Boot State Machine

The firmware operates as a strict linear state machine on every wake. There is no RTOS task structure — ESPHome's event loop drives all transitions.

```
                      ┌─────────┐
                      │  BOOT   │◄─── RTC wake / power-on
                      └────┬────┘
                           │ on_boot (priority 600)
                           │ sensor_readings_blocked = true
                           ▼
                  ┌─────────────────┐
                  │  WIFI CONNECT   │
                  │  (up to 3min    │
                  │   then reboot)  │
                  └────────┬────────┘
                           │ on_connect fires
                           ▼
                  ┌─────────────────┐
                  │  WAIT API READY │
                  │  deep_sleep     │
                  │  .prevent()     │
                  └────────┬────────┘
                           │ api.connected + 3s delay
                           ▼
                  ┌─────────────────────┐
                  │  CHECK ota_mode_ha  │
                  └──────┬──────────────┘
          ON ◄───────────┘───────────────► OFF
           │                               │
           ▼                               ▼
  ┌──────────────────┐         ┌──────────────────────┐
  │   AWAKE / OTA    │         │  unblock readings    │
  │  (readings       │         │  component.update()  │
  │   blocked)       │         │  delay 2s            │
  └────────┬─────────┘         └──────────┬───────────┘
           │                              │
           │ ota_mode_ha → OFF            │
           │ (on_state handler)           ▼
           │                    ┌──────────────────┐
           └───────────────────►│   DEEP SLEEP     │
             (blocked=true,     │   30 minutes     │
              no reading)       └──────────────────┘
```

> **Key invariant:** `sensor_readings_blocked` is never explicitly set to `true` — it starts `true` at boot and is only ever cleared. This means any unexpected code path (crash recovery, safe mode, fallback) leaves readings blocked by default.

---

## 5. Self-Heating Protection

### 5.1 Problem

The ESP32-C3's Wi-Fi radio and CPU generate heat that measurably raises the temperature seen by the BME280 during long awake sessions (OTA, debugging). Publishing readings during this period would corrupt HA history with systematically high temperature values, which in turn distorts derived humidity readings.

### 5.2 Solution Architecture

Three independent layers work together:

#### Layer 1 — Global Block Flag

```yaml
globals:
  - id: sensor_readings_blocked
    type: bool
    restore_value: no       # Never persisted across sleeps
    initial_value: 'true'   # Blocked from the very first instruction
```

`restore_value: no` is critical — it ensures the flag resets to `true` on every boot regardless of how the previous session ended (normal sleep, crash, OTA reset).

#### Layer 2 — Manual Sensor Trigger

```yaml
sensor:
  - platform: bme280_i2c
    id: bme280_sensor
    update_interval: never   # No auto-timer; must be triggered explicitly
```

The sensor never fires autonomously. `component.update: bme280_sensor` is called only once, from the `on_connect` automation, only on the normal-mode branch (switch OFF).

#### Layer 3 — Filter Lambda Guards

Each BME280 sub-sensor has a filter lambda that returns `NAN` while the flag is set:

```cpp
if (id(sensor_readings_blocked)) {
    ESP_LOGW("bme280", "Reading BLOCKED");
    return NAN;   // ESPHome discards NAN — no HA state update
}
return x;  // (+ calibration offset where applicable)
```

`NAN` propagates through ESPHome's filter pipeline and is silently dropped — the HA entity retains its last-known-good state rather than receiving a corrupted value.

### 5.3 Race Condition Eliminated

The previous design checked `ota_mode_ha.state` directly in the lambda. Because `ota_mode_ha` is a `homeassistant` binary sensor, its state is pushed from HA *after* the API handshake — it starts as `false` (unknown) at boot. Any sensor read that fired during this window would see `state == false` and publish unchecked.

The global flag approach removes this dependency: the block decision is made **before** any HA state is received.

---

## 6. Sleep / Wake Cycle

### 6.1 Timing

| Phase | Duration | Notes |
|---|---|---|
| Boot to Wi-Fi connected | ~3–6 s | Depends on AP response |
| Wi-Fi to API connected | ~1–2 s | Noise handshake |
| Delay (log streaming) | 3 s | Allows remote log clients to attach |
| BME280 measurement | ~0.1 s | 16x oversampling |
| Publish delay | 2 s | API state propagation to HA |
| **Total awake (normal)** | **~9–13 s** | |
| Deep sleep | 1800 s (30 min) | RTC-timer wakeup |

### 6.2 Fail-Safe Hardware Timer

`run_duration: 30s` in `deep_sleep` is a **hardware-level watchdog**. If the `on_connect` automation never fires (no Wi-Fi, no API), the device sleeps unconditionally at 30 seconds. This prevents battery drain from a permanently awake device in a network outage.

The `on_connect` automation calls `deep_sleep.prevent` as its first action, which cancels this hardware timer as soon as connectivity is confirmed.

### 6.3 Wi-Fi Reboot Safety

```yaml
wifi:
  reboot_timeout: 3min
```

If Wi-Fi does not connect within 3 minutes, ESPHome reboots the device. Combined with `run_duration: 30s`, this creates a two-layer protection:
- 30 s → hardware sleep (if connected but API fails)  
- 3 min → full reboot (if Wi-Fi itself fails to associate)

---

## 7. OTA & Safe Mode

### 7.1 OTA Flow

1. User toggles `input_boolean.keep_sensors_awake` **ON** in HA
2. Device remains awake (next wake cycle detects switch ON via `ota_mode_ha`)
3. User flashes via `esphome run TempHumidPres.yaml` (OTA over Wi-Fi) or USB
4. User toggles switch **OFF**
5. `on_state` handler fires → `deep_sleep.allow` + `deep_sleep.enter`
6. Next boot: normal cycle resumes (readings unblocked on that boot)

> **Note:** `sensor_readings_blocked` is **not** cleared in the `on_state` OFF path. The device goes straight to sleep without reading — after an OTA session the sensor may still be warm. Readings resume cleanly on the following boot.

### 7.2 Safe Mode

```yaml
safe_mode:
  reboot_timeout: 1min
```

ESPHome's built-in safe mode tracks consecutive boot failures. If the device fails to reach a stable state within 1 minute for 10 consecutive boots, it enters safe mode (OTA-only, no user code). The 1-minute window (down from the default 5 minutes) was chosen to minimise battery drain during a bad-firmware recovery scenario.

---

## 8. Hardware Abstraction — Sensor Power Rails

The BME280 is powered via GPIO-controlled rails rather than direct connection to 3.3V/GND:

```yaml
switch:
  - platform: gpio
    pin: GPIO20        # → BME280 VCC
    restore_mode: ALWAYS_ON

  - platform: gpio
    pin: GPIO10        # → BME280 GND
    restore_mode: ALWAYS_OFF
```

**Rationale:** Software-controlled power rails allow future power-cycling of the sensor (e.g., to clear a hung I²C bus or save power with the sensor off during sleep). `restore_mode: ALWAYS_ON/OFF` ensures the rails are at the correct level immediately at boot, before any automation runs.

> ⚠️ GPIO8 (SDA) and GPIO9 (SCL) are ESP32-C3 strapping pins. No external pull-down resistors should be attached to these pins, as this risks overriding the strapping values at boot.

---

## 9. Sensor Calibration

Calibration offsets are applied in the BME280 filter lambda, on the same line as the block check:

| Channel | Offset | Location in code |
|---|---|---|
| Temperature | `+0.0` (identity, placeholder) | `return x;` |
| Humidity | `+4.5 %` | `return x + 4.5;` |
| Pressure | `round(1)` then identity | `round` filter + `return x;` |

To recalibrate, update only the `return` expression in the relevant lambda. The block check (`if (id(sensor_readings_blocked)) return NAN;`) must remain as the first statement.

---

## 10. Security

| Mechanism | Implementation |
|---|---|
| API traffic encryption | Noise protocol (`api.encryption.key` from `secrets.yaml`) |
| OTA authentication | Password-protected (`ota.password` from `secrets.yaml`) |
| Fallback AP | Password-protected (`wifi.ap.password` from `secrets.yaml`) |
| Secrets management | `secrets.yaml` excluded from git via `.gitignore` |

---

## 11. Known Constraints and Limitations

| Constraint | Impact | Mitigation |
|---|---|---|
| GPIO8/GPIO9 are strapping pins | Risk of boot failure with pull-downs | Documented; no external resistors |
| No sensor read on first boot after OTA | One 30-min gap in HA history after each OTA session | Intentional — prevents warm readings |
| `run_duration` cuts session if API slow | Device may sleep before HA state is received | `deep_sleep.prevent` cancels it once Wi-Fi connects |
| Single firmware file | No A/B partition, no rollback binary | Safe mode provides re-flash window |
| I²C address fixed at 0x76 | Cannot use a second BME280 at 0x77 without hardware change | SDO pin → GND sets 0x76 |

---

## 12. Future Extension Points

| Feature | Suggested Approach |
|---|---|
| Sensor power-cycle on I²C hang | Add `switch.turn_off: pwr_pin_sensor` → delay → `switch.turn_on` in a recovery automation |
| CO₂ sensor (SCD40, UART) | Add `uart:` block + `scd4x` platform; extend block flag to cover new sensor |
| Button wake | Uncomment `wakeup_pin` in `deep_sleep`, add a `binary_sensor` for the pin |
| Variable sleep duration from HA | Use a `number` helper in HA + `homeassistant.number` entity; pass value to `deep_sleep.sleep_duration` lambda |
| Battery voltage monitoring | Add ADC sensor on a free GPIO with a voltage divider |
