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

---

## 4. All Out Ultra Power+ mosquito vaporizer on/off

- **Phase:** 3
- **Status:** DECIDED — switch its mains power with a Tuya Wi-Fi smart plug.
- **Context:** plug-in 230 V liquid vaporizer. No on/off switch; it is "on" whenever
  powered (only a low/high heat knob). Controlling it = switching mains. No need to open
  the device. A servo/motor on the knob was considered and rejected (fragile, pointless).

- **Decision:** **Tuya Wi-Fi smart plug**, same integration path as the bulb + switches.
  - Plug the All Out into it, leave the knob on medium.
  - Comes through `tuya-local` alongside the bulb; name the entity
    `switch.mosquito_repellent`; add a toggle to the PWA.
  - Set the plug's **power-on state = off** (or "remember") so a power blip does not
    silently re-enable it.
  - HA schedules it: on at dusk / off at dawn, or tie to room presence (LD2410, Phase 4).
  - Heater load ~5-7 W — any plug/relay handles it. Do NOT rapid-cycle; on/off only.

- **Fallback (only if everything must be one ESP32 box):** 5 V opto-isolated relay module
  on an ESP32 GPIO (ESPHome `switch` platform `gpio`), contacts breaking the live wire to
  a socket. Requires proper mains practice — fuse, insulation, strain relief, mains vs
  low-voltage separation. Preferred to avoid; the smart plug is the same effort as adding
  another WiFi switch with none of the risk.

---

## 5. Room LED strip (IR-remote RGB strip)

- **Phase:** 1-2 (IR learning), PWA panel any time
- **Status:** DECIDED — learn its IR remote with the same ESP32 blaster. Zero new hardware.
- **Context:** generic RGB LED strip with a small IR remote (power, brightness +/-,
  fixed colours, flash/fade/strobe/smooth modes). The receiver eye is on the strip's
  inline controller box.

- **Decision:** treat it exactly like the AC / LG monitor:
  1. Point the strip remote at the ESP32's IR receiver (GPIO15), press each button,
     capture the raw/NEC code from the ESPHome logs.
  2. Add one `template` `button` per function, transmitting via `ir_ac` (or a 3rd IR LED
     if the controller is far from the AC-aimed LED — cheap to add on another GPIO).
  3. Entities: `button.led_strip_power`, `_bright_up`, `_bright_down`, `_white`, `_red`,
     `_green`, `_blue`, `_flash`, `_fade` (already wired in `pwa/index.html`).

- **Caveats:**
  - IR is one-way, no state feedback — PWA buttons are fire-and-forget (no on/off
    indicator). Same as the AC panel.
  - Needs line-of-sight from an IR LED to the controller's receiver eye.
  - Optional future upgrade (not now): replace the controller with an ESP32 running WLED
    for true colour/brightness/effects + real state. Keeps to the "local, no rewrite"
    principle but is a Phase 5+ nicety.

### Note on entry #4 price
Tuya Wi-Fi smart plug came to **~Rs 700** (not the Rs 300-600 first estimated).

---

## 6. Orchestration approach (build vs runtime)

- **Phase:** all
- **Status:** DECIDED.
- **Two layers, lightest option each:**
  - **Runtime orchestration = Home Assistant native automations / scripts.**
    HA is already the orchestrator. **Node-RED is deferred** - it adds a second Node.js
    process on the old Android phone that also runs HA Core in Termux, against the
    "keep the phone light" principle in PLANNING.md. Reconsider Node-RED only if
    automation flows get genuinely complex (multi-branch, lots of state).
  - **Build orchestration = Task (`Taskfile.yml` at repo root).**
    Single cross-platform binary, YAML syntax like the rest of the repo, works in
    PowerShell. Wraps: `esphome:phase1` (flash), `esphome:logs` (IR learning),
    `esphome:compile`, `pwa:serve`, `yaml:lint`, `ble:capture`, `todo`.
    Install: `winget install Task.Task`; deps `pipx install esphome`.
- **Not chosen:** Node-RED (resources), n8n (overkill), Ansible (single phone, manual
  setup is documented and fine), Docker Compose (HA Core runs bare in Termux, not
  containerised).

---

## 7. Room door lock system

- **Phase:** 3-4 (new; own mini-project)
- **Status:** DECIDED (2026-08-28) — **servo throws the existing bolt**, battery-powered
  node (no mains near the door). Assumes the door has an **aldrop / tower bolt** — confirm.
- **Hard safety rules (non-negotiable, apply to every option):**
  1. **Mechanical egress always.** The inside handle / thumbturn must retract the bolt
     regardless of electronics, power, or hub state. A bedroom door that can trap you is
     a fire hazard. No purely-electronic latch on the inside.
  2. **Lock logic lives on the ESP32 (ESPHome), not the phone hub.** HA Core on an old
     Android phone is not reliable enough to sit between you and a locked door. It works
     if HA is down.
  3. **LAN only.** Never expose lock control to the internet.
  4. If electric strike / maglock: small battery/UPS so a power cut doesn't lock or
     unlock unexpectedly. Prefer **fail-safe** (unlocks on power loss) for a bedroom.
  5. Auto-lock on "room empty" (LD2410) only with a long delay + easy manual unlock -
     lock-out risk.

### Chosen design — servo-actuated bolt, battery node

- **Actuator:** **MG996R metal-gear servo** (~10-12 kg-cm, ~Rs 300) on a 3D-printed /
  metal bracket next to the aldrop, with a **lever arm + slotted linkage** to the bolt
  knob. Slotted/loose linkage is deliberate: the bolt must still slide **by hand** with
  the servo unpowered (safety rule #1). If the aldrop is stiff, upgrade to a 25 kg-cm
  servo or a worm-drive linear actuator (self-locking, holds position with no power).
- **Servo power management:** ESPHome `servo` on an LEDC `output`; a **MOSFET (or the
  servo's own enable) cuts servo VCC between operations** so it draws ~0 idle and can't
  buzz/back-drive. Move -> hold 500 ms -> detach + power off.
- **Controller + power (no mains at the door):** small dedicated **ESP32** node with
  **2x 18650 + holder + TP4056/again a protected BMS**, or a 6 V 4xAA pack through a
  buck to 5 V. Servo stall is ~2 A so size the battery/wiring for it.
  - **Battery life is the weak point.** A WiFi-always ESP32 + occasional servo pulses
    drains a 2S 18650 pack in days, not weeks. Mitigations, best first:
    1. **Run a thin USB cable** along the top of the frame to the nearest socket — often
       the least-bad option; then it's just a normal always-on node.
    2. Add a **small 5-6 V solar panel** trickle-charging the pack (door faces a window?).
    3. ESP32 **deep-sleep + wake every ~10 s** to poll a retained MQTT/HA flag —
       accept up to ~10 s lock/unlock latency, get weeks of life.
    4. Accept a weekly recharge / swap.
  - Decision on which mitigation: deferred to build time; try (1) first.
- **State feedback:** reed switch on the frame -> `binary_sensor.room_door` (Open/Closed,
  ~Rs 30). Bolt locked/unlocked = servo commanded position (ESPHome `lock` tracks it);
  optionally a micro-switch at the bolt's locked end-of-travel for true confirmation.
- **Integration:** ESPHome template `lock` -> HA `lock.room_door` -> PWA. `pwa/index.html`
  already has the **Door** panel (Unlock / Lock + `s_lock` + `s_door` tiles, `lockDoor()`
  helper calling `lock/lock` | `lock/unlock`).
- **Automation:** manual from the PWA first. Auto-lock on "room empty" (LD2410) only later,
  with a long delay + the reed sensor confirming the door is shut, per safety rule #5.

- **Open question:** confirm the door bolt is an aldrop / tower bolt (servo throw only
  makes sense for a sliding bolt). If it turns out to be a knob or mortise deadbolt,
  revisit — options A/B/D from the earlier draft still apply.

### Earlier options (kept for reference if the door hardware turns out different)
- A. Retrofit smart deadbolt (Godrej/Yale/Qubo/Ultraloq), ~Rs 8-20k, keeps thumbturn.
- B. 12 V solenoid strike/drop bolt + ESP32 relay, ~Rs 1.5-3k, needs mains/12 V at frame.
- D. Maglock (fail-safe only), ~Rs 1.2-3k + relay + PSU, needs power at frame.

---

## 8. AC unit — Godrej 2025 1.5 Ton 5-Star Window Inverter

- **Phase:** 1-2 (IR)
- **Status:** model known; IR protocol to be identified at capture time.
- **Unit:** Godrej 2025, 1.5 T, 5-star, **window** inverter AC. Remote has temp, mode,
  fan speed, swing, **Turbo**, timer, sleep. ("Anti-dust filter", "anti-freeze
  thermostat" are mechanical features, not IR-relevant.)

- **Plan — try for a real thermostat entity first:**
  1. During Phase 1, with `remote_receiver: dump: all`, press a few remote buttons and
     watch the ESPHome log for a named decoder ("Received Coolix / Gree / ...").
     Indian Godrej window ACs are commonly **Coolix**- or **Gree**-family.
  2. **If a known protocol matches** -> use the matching `climate_ir_*` platform
     (or the `climate_heatpumpir` external component) -> HA gets a full
     `climate.room_ac` with setpoint, mode, fan. Best outcome; exact "set 24 C" works.
  3. **If nothing decodes** -> fall back to `transmit_raw` template `button`s per
     function: `button.ac_power_toggle`, `ac_temp_up`, `ac_temp_down`, `ac_mode`,
     `ac_fan`, `ac_swing`, `ac_turbo` (all already wired in `pwa/index.html`).
     Note: these ACs send the **full state** in every frame, so raw replay of a fixed
     "temp up" frame only steps from the state it was captured in — capture each temp
     if precision matters, or just live with relative stepping.

- **PWA:** AC panel now includes **Turbo**. If we land a `climate` entity instead of
  buttons, revisit the panel to show a setpoint slider (optional, bigger change).

- **Open question:** which decoder (if any) lights up on first capture -> decides
  climate_ir vs raw buttons.
