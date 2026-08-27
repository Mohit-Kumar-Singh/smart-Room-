# TODO / Open Questions

Things to resolve before or during the phase that needs them. Solutions land in
`decisions-log.md`; this file is just the running checklist.

## Info still needed

- [ ] **Wipro bulb** — exact model + originating app (Wipro Smart / Wipro Next /
      Smart Life). Determines white-only vs RGB command set and whether the PWA Bulb
      panel needs a colour row. (Phase 3, decisions-log #1)
- [ ] **Godrej aer** — confirmed battery-powered (app shows 99% refill gauge, "Connected /
      last sync"). Still need: distance from the planned ESP32 IR-blaster spot to the
      dispenser (decides whether a 2nd ESP32 BLE node is required). (Phase 3, #2)

## Capture / bench tasks

- [ ] **Godrej aer BLE capture** — full byte map for: Spray Now, On, Off, Interval
      10/20/40 min, Reset Refill; plus the NOTIFY characteristic for refill %. Method in
      decisions-log #2. Known so far: service `6E400000-B5A3-F393-E0A9-E50E24DCCA9E`,
      write char `6E400003-...`, spray-now payload from community guide.
- [ ] **AC + LG monitor IR codes** — learn via ESPHome `remote_receiver` dump, paste into
      template buttons. (Phase 1-2)

## Research spikes (deferred)

- [ ] **Phone-as-appliance** — reuse the HA phone as media player + comms endpoint
      (white noise / music, camera+mic feed, "accept call" for a live VC feed) driven
      from the PWA, via screen-mirror + input injection. Options in decisions-log #3.
      Phase 6 stretch, do not block phases 1-4.
