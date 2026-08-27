# Decisions & Ready-Solutions Log

Running log of sub-problems worked out ahead of time. When we actually reach that
phase, the entry here should be enough to execute without re-thinking.

Format per entry: **Problem / Phase / Status / Decision / Ready steps / Open questions / Caveats.**

---

## 1. Wipro WiFi bulb integration

- **Phase:** 3
- **Status:** decided (local path), blocked on bulb model for the command set
- **Decision:** Wipro Smart / Wipro Next app is a **Tuya white-label**. Use a **local**
  integration, not cloud Tuya - the hub is an offline-first old phone, cloud control
  dies when the internet drops.
  - Primary: **`tuya-local`** ("Local Tuya" by make-all) via HACS - actively maintained,
    has a Wipro bulb profile, no cloud calls after setup.
  - Fallback: official HA **Tuya** cloud integration (Settings -> Devices & Services ->
    Add Integration -> Tuya, log in with the Wipro app account). Zero key extraction,
    but cloud-dependent + ~1-2 s latency.

- **Ready steps:**
  1. Pair the bulb via the Wipro/Smart Life app. **2.4 GHz WiFi only** - phone must be
     on the 2.4 GHz SSID during pairing.
  2. Reserve a **static DHCP lease** for the bulb in the router (local control needs a
     stable IP).
  3. Extract `device_id` + `local_key`: `pip install tinytuya` then
     `python -m tinytuya wizard`. It walks through creating a free iot.tuya.com cloud
     project, linking the app account, and writes all device ids + keys to
     `devices.json`. Cloud is touched **once**, here only.
  4. Install **HACS** on Home Assistant, add **Local Tuya / tuya-local**.
  5. Add the bulb: IP + device_id + local_key. DP mapping auto-detects for known Wipro
     profiles: on/off, brightness (usually DP 22, scale 0-1000), color-temp or RGB by
     model.
  6. Name the entity **`light.wipro_bulb`** - hardcoded in `pwa/index.html`
     (`toggleEntity('light.wipro_bulb')`, `setBrightness('light.wipro_bulb', ...)`).
     Rename in HA or update the PWA.
  7. Test: toggle + Full/Dim/Night brightness buttons from the PWA.

- **Open questions:**
  - Exact bulb model + originating app (Wipro Smart / Wipro Next / Smart Life)?
    - Tunable white (9W/12W "Smart White") -> on/off + brightness + color-temp
    - RGB ("Smart Color" / Garnet RGB) -> also hue/saturation/effects; PWA Bulb panel
      needs a colour row added.
  - `local_key` **rotates if the bulb is re-paired** in the app - re-run the wizard if
    that happens.

---

## 2. Godrej aer Smart Matic Kit (Bluetooth room freshener)

- **Phase:** 3
- **Status:** decided (two paths documented), execute at Phase 3
- **Context:** BLE-only, pairs to the "Godrej aer Smart" app. **No native HA
  integration.** A `bluetooth_proxy` alone does NOT expose a spray button.

- **Decision:** plan for **Path A** (hardware button tap). Attempt **Path B** (BLE replay)
  first only if opening the case is unacceptable.

### Path A - hardware button tap (primary, robust)
Solder two wires across the dispenser's manual **boost/spray** button, drive it from an
ESP32 GPIO via a transistor or opto-isolator, expose as an ESPHome `button`. HA handles
scheduling.
- Pros: guaranteed, no reverse-engineering, no BLE auth fight, ESP32 stays where the IR
  LEDs want it.
- Cons: opens the case (warranty); lose the app's own schedule (HA replaces it).

### Path B - replay the BLE GATT write (cleaner, risky)
1. Android: Developer Options -> enable **Bluetooth HCI snoop log**.
2. Open the Godrej app, press **spray** once. Pull `btsnoop_hci.log`, open in Wireshark.
3. Find the **ATT Write Request** - record the characteristic UUID + value bytes.
   Capture the whole sequence (connect -> any auth write -> spray write).
4. On the ESP32 add `esp32_ble_tracker` + `ble_client` (ESP32 becomes a BLE **client**,
   not just a proxy) and a template `button` calling `ble_client.ble_write` to that
   characteristic with the captured bytes.
- Open risks:
  - BLE devices accept **one connection** - once the ESP32 owns it, the app can't
    connect (and vice versa). Forget the device in the app after cutover.
  - If the spray command is protected by a rolling / bonded auth key, static replay
    fails -> fall back to Path A.
  - BLE range ~5-10 m. If the dispenser is far from the IR-blaster location, add a
    **second ESP32** (~Rs 450) as a dedicated BLE node near the freshener.
  - Dispenser may sleep / drop BLE to save battery -> few-seconds reconnect latency.

- **PWA wiring:** `pwa/index.html` already calls `fireButton('button.freshener_spray')`
  and `toggleEntity('switch.freshener_auto')` - name the HA entities to match, or update
  the PWA.

- **Open questions:**
  - Is the dispenser mains- or battery-powered? (affects BLE sleep behaviour + where the
    ESP32 must sit)
  - Distance from the planned ESP32 IR-blaster spot to the freshener?
