# Smart Room Remote

A DIY universal remote + room sensor system: control AC, LG monitor, a smart bulb,
WiFi switches, a smart plug, an IR LED strip, a servo door lock, and a BLE room
freshener from a custom iPhone PWA — backed by
Home Assistant Core (running on a reused old Android phone), with an ESP32 doing
IR + Bluetooth bridging.

See `PLANNING.md` for the full phased build plan, hardware list, and cost breakdown.

## Repo structure
- `esphome/` — ESP32 firmware config (IR transmit/receive, BLE proxy, room sensors)
- `homeassistant/` — Home Assistant integration setup notes (Tuya, IR, BLE)
- `pwa/` — the installable iPhone remote control web app
- `PLANNING.md` — phase-by-phase build plan with hardware + cost per phase
- `Taskfile.yml` — build commands (flash, logs, serve PWA, lint); runner: https://taskfile.dev
- `docs/` — `hardware-shopping-list.md` (Amazon.in parts + board rationale),
  `decisions-log.md` (pre-worked solutions per sub-problem), `todo.md` (open questions +
  bench tasks); circuit/wiring diagrams and enclosure design coming next

## Status
Planning complete; full phase-by-phase checklist in `docs/todo.md`.
Phase 1 config (`esphome/phase1-ir-poc.yaml`) is ready to flash — hardware not yet in hand.
