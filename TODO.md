# HA Sensor Improvements TODO

## Critical Logic Fixes
- [x] **Fix Wi-Fi Disconnect Reboot Loop:** The `on_disconnect` block currently logs that it is rebooting after 3 minutes, but it does not actually trigger a reboot. 
  - **Solution:** Delete the entire `on_disconnect` block and add `reboot_timeout: 3min` to the `wifi:` component. This allows ESPHome to manage the fallback and reboot natively.

## Future Considerations (Optional)
- [x] **Deep Sleep Timer Dependency:** `deep_sleep` currently triggers 30 seconds after a successful Wi-Fi connection. If the AP is down, the device will stay permanently awake, draining the battery until the Wi-Fi reconnects.
  - **Alternative Solution:** Use `run_duration: 35s` directly inside the `deep_sleep:` block to enforce a strict hardware uptime limit regardless of network state.
