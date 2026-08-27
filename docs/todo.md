# TODO — phase by phase

Master checklist. Pre-worked solutions live in `decisions-log.md`; hardware picks in
`hardware-shopping-list.md`. Tick items as done and push.

Legend: `[ ]` todo · `[~]` in progress · `[x]` done · **(sw)** = doable now, no hardware.

---

## Phase 0 — Procurement & prep (no hardware needed)

- [ ] Order Phase 1 parts (see `hardware-shopping-list.md`): ESP32 38-pin CP2102,
      breadboard + jumpers, IR receiver modules, IR emitter LEDs, 2N2222 x10, resistor kit
- [ ] Micro-USB **data** cable on hand (not charge-only)
- [ ] Install CP2102 driver on the Windows PC (usually auto; else Silicon Labs VCP)
- [x] **(sw)** Trim ESPHome config to IR-only for Phase 1 -> `esphome/phase1-ir-poc.yaml`
- [x] **(sw)** Build orchestration: `Taskfile.yml` (decisions-log #6)
- [ ] Install Task (`winget install Task.Task`) + `pipx install esphome`
- [ ] Measure distance: planned ESP32 spot -> Godrej freshener (decides 2nd ESP32 BLE node)
- [ ] Check Wipro bulb model + which app (white vs RGB) — `decisions-log.md` #1

## Phase 1 — Prove IR works (~1 day, ~Rs 500)

- [ ] Flash `phase1-ir-poc.yaml` via https://web.esphome.io over USB
- [ ] `secrets.yaml` from `secrets.yaml.example`, fill WiFi (keys auto-generated)
- [ ] Wire IR receiver OUT -> GPIO15, VCC/GND
- [ ] Wire IR LED -> 2N2222 -> GPIO4 (100-220R series, 1k base)
- [ ] Open ESPHome logs, point real AC remote at receiver, capture **AC power** raw code
- [ ] Paste code into the `AC Power` template button, fire from the device web page
- [ ] **Decision gate:** IR transmit reliable at real range? If yes -> Phase 2

## Phase 2 — Full IR + real hub (~2 days)

- [ ] Prep old Android phone: Developer Options, "don't optimize" Termux, Autostart on
      (MIUI/ColorOS/FuntouchOS), keep plugged near router
- [ ] Termux from **F-Droid** (not Play Store) -> `pkg install python git` -> `pip install homeassistant`
- [ ] Termux:Boot add-on so `hass` auto-starts on reboot
- [ ] Static DHCP leases for the phone AND the ESP32
- [ ] HA onboarding at `http://<phone-ip>:8123`; install ESPHome add-on / add the device
- [ ] Add 2nd IR LED -> GPIO16 (LG monitor); promote config to full `room-remote-bridge.yaml`
- [ ] Learn remaining codes: AC temp +/-, mode, fan, swing; monitor power/input/vol
- [ ] Learn **LED strip** remote codes (power, bright +/-, colours, flash/fade); add
      template buttons `button.led_strip_*` — already wired in the PWA. `decisions-log.md` #5
      (add a 3rd IR LED on another GPIO if the strip controller is off-axis from the AC LED)
- [ ] Create HA long-lived access token
- [x] **(sw)** PWA: switch settings from in-memory to `localStorage` (`pwa/index.html`)
- [ ] **(sw)** PWA: add `icon-192.png` / `icon-512.png` (referenced by manifest, missing)
- [ ] Deploy PWA (static host or HA `www/`), add to iPhone home screen, enter URL + token
- [ ] Verify AC + monitor controls from the PWA
- [ ] Bonus: IP Webcam on the phone -> HA Generic Camera

## Phase 3 — WiFi + Bluetooth devices (~1 day)

- [ ] Wipro bulb: `pip install tinytuya` -> `python -m tinytuya wizard` -> device_id + local_key
- [ ] Install HACS; add **tuya-local**; add bulb by IP + id + key
- [ ] Name entity `light.wipro_bulb`; test toggle + brightness from PWA
- [ ] WiFi switches: same Tuya path; entities `switch.room_switch_1` / `_2`
- [ ] Mosquito vaporizer (All Out Ultra Power+): Tuya Wi-Fi smart plug, knob on medium,
      plug power-on state = off; entity `switch.mosquito_repellent`; add PWA toggle.
      `decisions-log.md` #4
- [ ] If bulb is RGB: add a colour row to the PWA Bulb panel
- [ ] **Door lock** — `decisions-log.md` #7. First decide: current door hardware
      (aldrop / knob / mortise+deadbolt / lever+cylinder), budget (retrofit smart lock
      ~Rs 10k vs DIY 12 V solenoid ~Rs 2k), power reachable at the frame. Then:
  - [ ] Fit lock mechanism (keep mechanical inside egress — mandatory)
  - [ ] Reed switch on frame -> `binary_sensor.room_door`
  - [ ] ESPHome `lock` -> `lock.room_door` (logic on ESP32, not the phone; LAN only)
  - [ ] PWA Door panel already wired (`lockDoor()`, `s_lock`, `s_door`)
- [ ] Godrej aer (full BLE) — `decisions-log.md` #2:
  - [ ] nRF Connect GATT dump; confirm write char `6E400003`, find NOTIFY char
  - [ ] HCI snoop capture: On, Off, Interval 10/20/40, Spray Now, Reset Refill
  - [ ] Check for init/auth write on connect
  - [ ] ESPHome `esp32_ble_tracker` + `ble_client`; buttons + interval select + refill sensor
  - [ ] Cutover: forget device in Godrej app
  - [ ] PWA: entities `button.freshener_spray`, `switch.freshener_auto`, interval, refill %

## Phase 4 — Room sensors (~1 day, ~Rs 900)

- [ ] Buy: BME280 (verify humidity, not BMP280), MQ-135, KY-037, LD2410C
- [ ] Wire: BME280 I2C SDA/SCL GPIO21/22 (addr 0x76); KY-037 AO -> GPIO34;
      MQ-135 AO -> GPIO35; LD2410 TX/RX -> GPIO17/18
- [ ] Add the external `ld2410` component to the ESPHome config
- [ ] MQ-135 burn-in 24-48 h, then set calibration baseline
- [ ] Confirm all six sensor tiles populate in the PWA (`sensor.room_*`, `binary_sensor.room_*`)

## Phase 5 — Diagrams & enclosure

- [ ] Circuit diagram (Fritzing or KiCad) for the ESP32 + IR + sensor build
- [ ] Wiring diagram with GPIO/pin callouts
- [ ] Enclosure: project box or 3D-printed; IR LED windows aimed at AC + monitor;
      vents for MQ-135/BME280; radar has clear plastic in front

## Phase 6 — Phone-as-appliance (stretch, do not block 1-4)

- [ ] HA <-> phone over ADB; `adb tcpip 5555` persisted via Termux:Boot; LAN only
- [ ] HA scripts: play audio / white noise, launch IP Webcam, "accept call" tap macro
- [ ] PWA: add a "Phone" panel exposing those scripts
- [ ] Later: ws-scrcpy server (on a non-phone machine) for browser screen mirror; link from PWA

## Cross-cutting / housekeeping

- [ ] Keep `decisions-log.md` current as sub-problems get solved
- [ ] `secrets.yaml` must stay gitignored (already in `.gitignore`)
- [ ] Update README **Status** at the end of each phase
- [ ] Add `docs/` circuit/wiring diagrams when Phase 5 starts
