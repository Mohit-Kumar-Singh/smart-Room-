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
- **Status:** DECIDED — **full Bluetooth mode** (ESP32 as BLE client). User does not care
  about warranty; the hardware-tap fallback is dropped as primary. Bench capture pending.
- **Context:** BLE-only, pairs to the "Godrej aer Smart Matic" app
  (`com.godrejcp.aermatic`). No native HA integration. A `bluetooth_proxy` alone does NOT
  expose controls — the ESP32 must be a BLE **client**.
- **Device (from app screenshots):** battery/refill gauge (99%), "Connected / last sync
  Ns". One physical button on the unit cycles states; the app exposes discrete controls:
  - **Spray Now** — single spray
  - **On/Off** — toggle auto-spray mode
  - **Spray Interval** — cycles 10m / 20m / 40m
  - **Reset Refill** — resets the refill gauge to 100%
  - Smart Scheduler (app-side; HA replaces it)

### Known BLE protocol (from community guide, home-automation-india.github.io)
- Service UUID: `6E400000-B5A3-F393-E0A9-E50E24DCCA9E`
- Write characteristic: `6E400003-B5A3-F393-E0A9-E50E24DCCA9E`
- **Spray Now** payload (verify against our own capture):
  `bf 62 6d 54 18 68 62 6d 4e 18 9a 62 72 49 00 ff`
  (framing looks like `bf ... 00 ff`; other commands likely share it)
- Notify characteristic for refill %/state: unknown — likely `6E400002-...`, confirm by
  GATT dump.

### Capture plan (the "how")
1. **nRF Connect (Android):** connect to the unit, dump all services/characteristics.
   Confirm `6E400003` is WRITE / WRITE-NO-RESPONSE; find the NOTIFY char and subscribe.
2. **HCI snoop:** Developer Options -> enable *Bluetooth HCI snoop log*. In the app press,
   in a noted order: On, Off, Interval->10, ->20, ->40, Spray Now, Reset Refill (one each,
   pause between). Take a bug report, extract `btsnoop_hci.log`, open in Wireshark.
   Filter `btatt.opcode.method == 0x12 || btatt.opcode.method == 0x52`. Record the value
   bytes written to `6E400003` for each action.
3. **Check the connect sequence** for a fixed init/auth write before commands are
   accepted; if present, replay it on connect.
4. **ESPHome:** add `esp32_ble_tracker:` + `ble_client:` (freshener MAC). Template
   `button`s -> `ble_client.ble_write` to `6E400003` with each captured payload
   (Spray Now, On, Off, Reset Refill); a `select` (or 3 buttons) for interval 10/20/40.
   `ble_client` `sensor`/`text_sensor` on the NOTIFY char -> parse refill % + state.
5. **Cutover:** remove the device from the Godrej app so the ESP32 keeps the sole BLE
   link; ESP32 holds a persistent auto-reconnect connection.

### Caveats
- BLE = **one connection**. App and ESP32 can't both hold it. Forget it in the app.
- If a command turns out to be protected by rolling/bonded auth (static replay fails),
  fall back to the **hardware button tap**: transistor/opto across the unit's physical
  button on an ESP32 GPIO. Kept as plan B only.
- Range ~5-10 m; battery unit. If it's far from the ESP32 IR-blaster spot, add a
  **2nd ESP32** (~Rs 450) as a dedicated BLE node — see `docs/todo.md`.
- Unit may drop BLE to save battery -> few-seconds reconnect latency.

### PWA wiring
`pwa/index.html` already calls `fireButton('button.freshener_spray')` and
`toggleEntity('switch.freshener_auto')`. Add entities for interval + refill %; name HA
entities to match or update the PWA.

---

## 3. Phone-as-appliance — reuse the HA phone as media + comms endpoint

- **Phase:** 6 (stretch) — research done, **do not block phases 1-4**
- **Goal:** drive the always-on HA phone from the PWA to: play white noise / music, expose
  its camera + mic as a live feed, and "accept a call" so an app's video-call (Telegram /
  Instagram / Meet) becomes an ad-hoc live view of the room.
- **Approaches (can combine):**
  - **A. ws-scrcpy / ws-scrcpy-web** (github.com/NetrisTV/ws-scrcpy, successor
    bilbospocketses/ws-scrcpy-web): a Node server pushes Genymobile's `scrcpy-server` over
    ADB and multiplexes video+audio+control onto one WebSocket; browser decodes via
    WebCodecs. Full screen mirror + tap/type from a browser tab -> link/embed from the
    PWA. Audio forward needs Android 11+. Run the Node server on another always-on machine
    if the phone CPU is tight.
  - **B. HA ADB service** (`androidtv`/adb integration talks to any ADB device): HA sends
    `input tap x y`, `input keyevent`, `input text`, `am start <intent>`. Wrap as HA
    scripts/buttons -> PWA calls them. No video, lightweight, good for concrete actions
    ("play playlist", "answer call"). **Start here.**
  - **C. On-device accessibility automation** (MacroDroid / Tasker+AutoInput) triggered by
    HA webhook — reliable in-app taps without coordinate guessing.
  - **Camera/mic feed:** IP Webcam (already planned, Phase 2 bonus) gives one-way
    camera+audio -> HA Generic Camera. Two-way "accept call" has no clean API — it's a
    coordinate tap on the accept button (option B/C) + the phone's own speaker/mic.
- **Enablement:** USB debugging on; `adb tcpip 5555` — re-arm after every reboot via a
  Termux:Boot script (or root). **LAN only — never expose ADB to the internet.**
- **Risks:** the phone is also the HA server — scrcpy H.264 encode + HA + camera stream
  may overload an old CPU (heat, battery, lag). Coordinate taps break on app updates.
  ADB-over-TCP resets on reboot.
- **Recommendation:** begin with **B** (HA ADB scripted actions) for real needs like white
  noise; add **A** (ws-scrcpy) later only if full remote screen is wanted.
