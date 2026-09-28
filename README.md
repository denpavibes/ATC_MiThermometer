# ATC_MiThermometer — Wireless Display Edition

A specialized fork of [pvvx's ATC_MiThermometer](https://github.com/pvvx/ATC_MiThermometer) custom firmware for Telink-based BLE thermometers (such as the Xiaomi Mijia LYWSD03MMC), optimized for use as a dedicated wireless remote display via ESPHome's [`pvvx_mithermometer`](https://esphome.io/components/display/pvvx_mithermometer.html) display platform.

---

> [!WARNING]
> **Stock Firmware Version 2.1.1_0159 & Hardware Revisions Notice:**
> - [Stock firmware version `2.1.1_0159` requires registration in Mi Home and obtaining bind keys/IDs](https://github.com/pvvx/ATC_MiThermometer/issues/602#issue-2786915630) before it can be flashed OTA.
> - Xiaomi LYWSD03MMC versions **B1.5 / B1.6** (manufactured since 2025.03) are not recommended for purchase due to [higher current consumption](https://github.com/pvvx/ATC_MiThermometer/issues/664#issuecomment-3092344109) and [low-contrast LCD displays](https://github.com/pvvx/ATC_MiThermometer/discussions/663#discussioncomment-13440647).
> - New LYWSD03MMC units with HW **B1.6** use original firmware that is incompatible with older HW versions. Follow the [upstream instructions](https://github.com/pvvx/ATC_MiThermometer/issues/602) for the initial OTA flashing procedure.

## Purpose & Key Modifications

In standard upstream firmware, external data sent via BLE command `0x22` (`CMD_ID_EXTDATA`) is treated as a secondary view in a periodic carousel, cycling back and forth between external data and local sensor readings every ~2.5 seconds. In addition, LCD refreshes were queued for the next periodic timer tick rather than applied immediately.

This fork turns the thermometer into an efficient, responsive wireless screen:

1. **Dedicated Remote Display Mode (`src/lcd.c`, `src/lcd.h`, `src/app.c`)**:
   - **Pre-Connection / Reset**: Until external data is received for the first time after a reset, the thermometer displays local sensor readings and the Bluetooth indicator stays off.
   - **Active Remote Session**: Once external data is received, the display strictly renders the remote values without cycling with local sensor readings or clock/battery screens. The Bluetooth indicator stays **steady ON** throughout the validity period (`vtime_sec`).
   - **Deep Sleep Optimization**: While valid external data is on screen, the MCU suppresses periodic ~2.45s LCD refresh wakeups, keeping the processor in deep retention sleep and conserving battery.
   - **Connection Lost / Stale Data Indication**: If the validity timer expires (e.g., ESP32 goes offline or Wi-Fi drops), the remote data **remains on screen** (avoiding sudden confusion with local room temperature) while the Bluetooth indicator **blinks at 1 Hz** to clearly signal a stale connection. Normal steady display and sleep resume immediately when a new packet arrives.

2. **Zero-Latency Screen Updates (`src/cmd_parser.c`)**:
   - Invokes `SET_LCD_UPDATE()` immediately when `CMD_ID_EXTDATA` is received over BLE.
   - Screen content redraws instantly upon transmission rather than waiting up to 2.5–5 seconds for the internal cadence timer.

3. **Dedicated Error Display ("Err")**:
   - Sending `-100` as the big number (e.g., `it.print_bignum(-100)` in ESPHome) displays `"Err"` across all supported LCD and E-Ink display models instead of a standard numerical value or `"Lo"`.
   - The small number field remains available to display numeric error codes (e.g., `it.print_smallnum(1)`).

4. **Modern CMake Build System**:
   - Replaced legacy Telink IoT Studio Eclipse configuration with a modern CMake toolchain (`tc32-elf-gcc`).
   - Clean VS Code build/clean tasks and submodule-based SDK dependency management.

---

## ESPHome Integration

Use the official ESPHome `pvvx_mithermometer` display component with an active `ble_client`.

### Recommended Pattern: Conditional Updates on Significant Changes

To maximize battery life and avoid unnecessary BLE traffic, set `update_interval: never` on the display component and trigger screen updates on-demand. Using a combination of `delta` and `heartbeat` filters ensures updates occur only on significant value changes or as a periodic keep-alive. A debouncing script with `mode: restart` and a 1-second delay prevents colliding back-to-back updates when multiple sensors report concurrently:

```yaml
esp32_ble_tracker:

ble_client:
  - mac_address: A4:C1:38:XX:XX:XX
    id: ble_display

display:
  - platform: pvvx_mithermometer
    id: my_display
    ble_client_id: ble_display
    update_interval: never # Updates triggered on-demand via script
    validity_period: 300s  # Must exceed the sensor heartbeat interval
    lambda: |-
      if (isnan(id(power_usage).state)) {
        it.print_bignum(-100); // Displays "Err" on screen
        it.print_smallnum(1);  // Error code
        it.print_sad();
        return;
      }
      it.print_bignum(id(power_usage).state);
      it.print_unit(pvvx_mithermometer::UNIT_NONE);
      it.print_smallnum(id(outside_temp).state);
      it.print_percent(false);
      it.print_happy(true);

# Debounce script: collapses near-simultaneous sensor changes into a single BLE connection
script:
  - id: trigger_display_update
    mode: restart
    then:
      - delay: 1s
      - component.update: my_display

# Example sensors triggering the display update
sensor:
  - platform: homeassistant
    id: power_usage
    entity_id: sensor.household_power
    filters:
      - delta: 10.0      # Only trigger on changes >= 10 W
      - heartbeat: 120s  # Keep-alive to prevent validity_period timeout
    on_value:
      then:
        - script.execute: trigger_display_update

  - platform: homeassistant
    id: outside_temp
    entity_id: sensor.outside_temperature
    filters:
      - delta: 0.5       # Only trigger on changes >= 0.5 °C
      - heartbeat: 120s
    on_value:
      then:
        - script.execute: trigger_display_update
```

### Best Practices for Wireless Display Use

- **Battery Optimization (CR2032)**:
  Each BLE connection draws ~8–12 mA during radio exchange. The conditional update pattern above keeps radio activity to a minimum, allowing the coin cell to last for months.
- **Heartbeat vs. Validity Margin**:
  Ensure your sensor `heartbeat` interval (e.g. 120s) is comfortably shorter than the display's `validity_period` (e.g. 300s) so static readings don't inadvertently trigger the blinking disconnected indicator. Note that the effective safety margin is also reduced by ESPHome's BLE reconnection time (typically 10–30s), so the true minimum gap is `validity_period - heartbeat_interval - ble_reconnect_time`. With the example values above that leaves ~150s of headroom, which is more than sufficient.
- **Device Config Settings**:
  In the web flasher configuration ([TelinkMiFlasher](https://pvvx.github.io/ATC_MiThermometer/TelinkMiFlasher.html)), make sure **"Show battery"** and **"Show clock"** are disabled so they do not periodically override the external display fields. You can also increase the internal measurement interval to conserve battery power.

---

## Building

### Prerequisites

- Telink TC32 Toolchain (`tc32-elf-gcc`, downloaded via the [Telink VS Code Extension](https://marketplace.visualstudio.com/items?itemName=telink.tlk))
- CMake $\ge$ 3.16 and Ninja / Make
- Python 3

### Command Line

```bash
git submodule update --init --recursive
cmake -B build -G "Ninja" -DCMAKE_TOOLCHAIN_FILE=cmake/tc32-toolchain.cmake
cmake --build build
```

---

## Acknowledgments & Upstream

- Original firmware and reverse engineering by [pvvx](https://github.com/pvvx/ATC_MiThermometer).
- Custom firmware foundations by [atc1441](https://github.com/atc1441/ATC_MiThermometer).
- ESPHome display platform implementation by [ESPHome contributors](https://esphome.io/components/display/pvvx_mithermometer.html).
