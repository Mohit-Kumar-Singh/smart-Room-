# Build Plan — Smart Room Remote

## Devices in scope
- LG monitor (controlled via IR, ThinQ app not used)
- AC (IR remote)
- Wipro smart bulb (WiFi, likely Tuya-based)
- WiFi switches (likely Tuya-based)
- Bluetooth room freshener spray (BLE)
- Room sensors: temperature, humidity, noise, air quality/"smell", motion/presence

## Architecture decision
**Raspberry Pi 4 (Home Assistant) + ESP32 (ESPHome) hybrid.**
- Pi 4 runs Home Assistant OS as the central brain/server — the PWA talks to this.
- ESP32 runs ESPHome as a satellite bridge: fires 2x IR emitters (AC + monitor),
  acts as a Bluetooth proxy for BLE devices, and hosts the room sensors.
- Rejected pure-ESP32-only plan because it doesn't scale to future features
  (voice control, automations, cameras) without a rewrite.
- Rejected raw Arduino/custom-protocol firmware in favor of ESPHome, since it
  auto-integrates with Home Assistant and avoids writing/maintaining a custom API.

## Phase 1 — Prove IR control works
**Cost: ~₹500 | Time: ~1 day**
- Hardware: ESP32, 1x IR LED + transistor + resistor, IR receiver (TSOP), breadboard/wires
- No Pi yet — flash ESPHome via the browser-based web installer over USB
- Learn one AC IR code, fire it from ESPHome's local device page
- Decision point: confirms IR learn/transmit works reliably before spending more

## Phase 2 — Full IR coverage + real hub
**Cost: ~₹6,500–7,000 | Time: ~2 days**
- Hardware: Pi 4 (4GB), SD card, case/fan, power adapter, 2nd IR LED+transistor (for monitor)
- Install Home Assistant OS, add the ESPHome device, learn remaining AC + LG monitor codes
- Deploy the PWA to iPhone home screen, control both devices from the app

## Phase 3 — WiFi + Bluetooth devices
**Cost: ~₹0 (no new hardware) | Time: ~1 day**
- Add Tuya integration in HA for the bulb + WiFi switches
- Sniff the freshener's BLE GATT command (nRF Connect app) and wire it up via
  the ESP32's Bluetooth proxy

## Phase 4 — Room sensors
**Cost: ~₹800–1,000 | Time: ~1 day**
- Hardware: BME280 (temp/humidity/pressure), KY-038 (noise), MQ-135 (air quality),
  LD2410 mmWave radar (presence/motion) — or plain PIR as a simpler fallback
- Wire into the same ESP32 (I2C for BME280, ADC pins for noise/air quality,
  UART for LD2410)
- Add live sensor readouts to the PWA

## Phase 5 (next) — Diagrams & product design
Not yet started. Planned: circuit/wiring diagrams for the ESP32 build,
enclosure/product design for mounting it in the room.

## Hardware ordering notes
- ESP32 IR emitters: 1 ESP32, 2 separate IR LEDs on different GPIOs
  (GPIO4 for AC, GPIO16 for monitor) rather than 2 separate boards — cheaper,
  simpler, one WiFi/BLE proxy still covers the whole room, since AC and
  monitor are in the same room (not far apart across walls).
- LG monitor: controlled via IR only, no ThinQ API integration (needs
  developer approval, overkill for this use case).
