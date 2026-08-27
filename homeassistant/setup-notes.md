# Home Assistant setup — device by device

Install Home Assistant OS on the Pi 4 first (Raspberry Pi Imager → "Home Assistant OS" → your SD card). Boot it, finish onboarding at `homeassistant.local:8123`.

## 1. ESP32 (ESPHome bridge)
- Settings → Add-ons → search "ESPHome" → install → open the ESPHome dashboard
- New device → paste `room-remote-bridge.yaml` content → install via USB the first time
- Once online, Home Assistant will auto-discover it (notification appears) → click "Configure"
- Your IR buttons and BLE proxy now show up as entities automatically

## 2. Wipro bulb + WiFi switches (likely Tuya)
- Settings → Devices & Services → Add Integration → search "Tuya" (or "Local Tuya" for fully local control, no cloud)
- Log in with the same account as your Smart Life / Tuya app — bulb and switches import automatically
- If bulb/switches don't show under plain "Tuya" app branding, open the app they came with and check Settings → About, it'll usually say powered by Tuya

## 3. LG monitor
- Skip ThinQ integration entirely (needs LG developer approval, overkill for a monitor)
- Control it purely via the IR codes you capture from the real remote — same method as the AC

## 4. AC (IR only)
- Capture codes using the ESPHome logs method (see LEARNING-IR-CODES.md)
- Common brands (LG, Voltas, Daikin) often have codes in Home Assistant's built-in `climate_ir` integrations too — worth checking before manually capturing every temperature step

## 5. Bluetooth freshener
- Once the ESP32's `bluetooth_proxy` is online, go to Settings → Devices & Services → Bluetooth
- If the freshener uses a documented BLE profile, HA may detect it directly
- If not (most cheap ones), you'll need the raw GATT write command sniffed via nRF Connect app, then trigger it using HA's `bluetooth_le` "ESPHome ble_client" component with a custom `ble_client.ble_write` action

## 6. Long-lived access token (for your PWA)
- Click your profile (bottom left) → scroll to "Long-Lived Access Tokens" → Create Token
- Save it somewhere safe — your PWA uses this to call HA's REST API
