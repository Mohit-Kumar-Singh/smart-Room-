# Smart Room Remote

A DIY universal remote + room sensor system: control AC, LG monitor, Wipro bulb,
WiFi switches, and a BLE room freshener from a custom iPhone PWA — backed by
Home Assistant Core (running on a reused old Android phone), with an ESP32 doing
IR + Bluetooth bridging.

See `PLANNING.md` for the full phased build plan, hardware list, and cost breakdown.

## Repo structure
- `esphome/` — ESP32 firmware config (IR transmit/receive, BLE proxy, room sensors)
- `homeassistant/` — Home Assistant integration setup notes (Tuya, IR, BLE)
- `pwa/` — the installable iPhone remote control web app
- `PLANNING.md` — phase-by-phase build plan with hardware + cost per phase
- `docs/` — `hardware-shopping-list.md` (Amazon.in parts + board rationale); circuit/wiring diagrams and enclosure design coming next

## Status
Planning complete. Phase 1 (basic IR proof-of-concept) not yet started.
