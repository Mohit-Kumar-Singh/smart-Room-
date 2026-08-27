# Running Home Assistant on an old Android phone (instead of a Raspberry Pi)

## Prep the phone
1. Settings → About phone → tap "Build number" 7x to enable Developer Options
2. Developer Options → enable "Stay awake" (screen stays on while charging — doesn't matter, you can turn the screen off manually, this just stops it from sleeping)
3. Settings → Battery → Battery Optimization → find Termux → set to "Don't optimize" / "No restrictions"
4. Keep the phone permanently plugged in near your router (WiFi signal matters more than distance to devices, since ESP32/bulb/switches talk to it over WiFi, not Bluetooth)
5. If it's Xiaomi/MIUI, Oppo, or Vivo: also check Settings → Apps → Termux → Autostart = ON, and look for a manufacturer-specific "battery saver" list to whitelist Termux in

## Install Termux
- Download from **F-Droid** (f-droid.org), NOT the Play Store — the Play Store build is outdated and breaks package installs
- Open Termux, run:
  ```
  pkg update && pkg upgrade
  pkg install python python-pip git
  ```

## Install Home Assistant Core
```
pip install homeassistant
```
(Takes a while on an old phone's CPU — this is normal, let it finish.)

## Run it
```
hass
```
First run creates its config at `~/.homeassistant/`. Once you see "Home Assistant started", open a browser on any device on the same WiFi and go to:
```
http://<phone's-local-IP>:8123
```
(Find the phone's IP: Settings → About phone → Status → IP address)

## Keep it running in the background
Termux gets killed by Android eventually unless you run it as a proper background service:
```
pkg install termux-services
sv-enable homeassistant   # if using termux-services with a hass service script
```
Simplest reliable approach: install the **Termux:Boot** add-on app (also from F-Droid) so `hass` auto-starts whenever the phone reboots, and keep a persistent notification open (Termux shows one automatically) — Android is far less likely to kill an app with an active foreground notification.

## Bonus: use the phone's camera too
- Install **IP Webcam** (Play Store, free) on the same phone
- Start the server in the app, note the stream URL it shows (e.g. `http://<ip>:8080/video`)
- In Home Assistant: Settings → Devices & Services → Add Integration → "Generic Camera" → paste that stream URL
- You now get room camera monitoring for ₹0 extra, using the same phone

## Known limitations vs. Raspberry Pi
- Android may still kill background processes unpredictably on some phones — check every few days initially
- Slightly slower than a Pi 4 for the first few weeks of setup/testing, though fine once stable
- If it becomes unreliable, moving to a Pi later is a config copy, not a rebuild — Home Assistant config transfers directly (`~/.homeassistant/` folder → Pi's config folder)
