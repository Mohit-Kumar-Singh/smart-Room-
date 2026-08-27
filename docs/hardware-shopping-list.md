# Hardware Shopping List (Amazon.in)

Prices, ratings and links captured **2026-08-27** (Amazon.in, Agra pincode). They will
drift — treat them as a guide, not a quote. Each pick is the listing with the best
rating x review-count that still has the *correct* part spec.

## Board choice — why ESP32 (not Uno / not Pi)

| Need | ESP32 devkit | Arduino Uno | Raspberry Pi |
|---|---|---|---|
| WiFi built-in | yes | no (needs shield) | yes |
| Bluetooth LE (freshener proxy) | yes | no | yes, but no ESPHome BLE-proxy stack |
| Runs ESPHome (auto HA integration, OTA) | native | can't (AVR) | no (ESPHome targets MCUs) |
| Precise 38 kHz IR carrier | hardware RMT peripheral | bit-banged | Linux timing jitter, unreliable |
| Analog inputs (noise, air quality) | multiple ADC pins | yes, but 5 V logic | none |
| Logic level vs sensors | 3.3 V (match) | 5 V (needs shifting) | 3.3 V |
| Cost (India) | Rs 300-600 | Rs 500-900 | Rs 1500-4500 |

- The Pi's original role here was the **hub** (Home Assistant) - already replaced by a
  reused old Android phone. As the IR/sensor node the Pi is the wrong tool.
- ESP8266 is rejected too: no Bluetooth, so it can't proxy the BLE freshener.
- Buy the **38-pin** WROOM-32 board (30-pin omits some needed GPIOs). Board type in
  ESPHome = `esp32dev`, which is what `esphome/room-remote-bridge.yaml` targets.
- Prefer the **CP2102** USB-UART chip (cleaner driver on Windows than CH340).
- Skip ESP32-**S3 / S2 / C3** boards - different pin map, the YAML won't match.

## Phase 1 - buy now (IR proof of concept)

| # | Part | Listing | Price | Rating |
|---|---|---|---|---|
| 1 | ESP32 38-pin (WROOM-32, CP2102) | Electrobot ESP-WROOM-32 CP2102 38-Pin - https://www.amazon.in/dp/B08VWG5JBQ | Rs 494 | 4.1* (60) |
|   | alt | Robocraze 38-Pin CP2102 - https://www.amazon.in/dp/B078MNG9D5 | Rs 631 | 3.8* (46) |
| 2 | Breadboard + jumper wires (MM/MF/FF) | Tishvi 3-in-1 jumpers + 840-pt breadboard - https://www.amazon.in/dp/B0GYM3S8VQ | Rs 355 | 4.0* (25) |
|   | cheaper split | Themisto 830 breadboard - https://www.amazon.in/dp/B0CGJDY5HL (Rs 158, 4.2*, 527) + Tishvi 240 jumper wires - https://www.amazon.in/dp/B0GBKB6B8N (Rs 450, 4.4*, 46) | | |
| 3 | IR receiver (learn codes) | INVENTO 5x VS1838B 38 kHz receiver modules - https://www.amazon.in/dp/B08182NJW9 | Rs 279 | 4.5* (28) |
|   | bare-sensor alt | MY TechnoCare TSOP1738 x3 - https://www.amazon.in/dp/B072HTN85B | Rs 272 | 4.4* (20) |
| 4 | IR emitter LED (the blaster) | Invento 25x 5mm 940nm IR emitter - https://www.amazon.in/dp/B07GD1JDR6 | Rs 279 | 4.2* (31) |
| 5 | NPN transistor (drives IR LED) | Electronic Spices 2N2222 x10 - https://www.amazon.in/dp/B0B88CRKSW | Rs 132 | 4.1* (56) |
| 6 | Resistor assortment (1/4 W) | AVS Components 500 pcs, 50 values - https://www.amazon.in/dp/B0D6LRHHVZ | Rs 199 | 4.5* (118) |

**Phase 1 subtotal: ~Rs 1,400-1,750.**
Also need a **micro-USB data cable** (not charge-only) - a phone cable usually works, no need to buy.

Wiring targets (from `esphome/room-remote-bridge.yaml`):
- IR LED via transistor -> **GPIO4** (AC), 2nd IR LED -> **GPIO16** (monitor)
- IR receiver OUT -> **GPIO15** (idles HIGH, fine on this strapping pin as input)
- Resistors needed: ~100-220 ohm (LED series), ~1 kohm (transistor base)

## Phase 3 - WiFi + BLE devices

| # | Part | Notes | Approx price |
|---|---|---|---|
| - | **Tuya Wi-Fi smart plug** (16A, with power monitoring optional) | for the All Out Ultra Power+ mosquito vaporizer - switch its mains. Any Tuya/Smart Life plug; comes through `tuya-local` with the bulb. Set power-on state = off. | Rs 300-600 |
| - | (maybe) **2nd ESP32 38-pin** | only if the Godrej freshener is >5-8 m from the main ESP32 - dedicated BLE node. Same board as Phase 1. | Rs 494 |

## Phase 4 - buy later, only when you reach sensors

| # | Part | Listing | Price | Rating | Note |
|---|---|---|---|---|---|
| 7 | BME280 (temp/humidity/pressure) | BME280 Digital Sensor Module (I2C/SPI) - https://www.amazon.in/dp/B07L84R8V4 | Rs 654 | 4.5* (8) | Many "BME280" listings are actually BMP280 (no humidity). Confirm humidity + I2C addr 0x76 (matches the YAML). |
| 8 | MQ-135 (air quality) | Robocraze MQ-135 - https://www.amazon.in/dp/B07MY5PB8L | Rs 257 | 4.1* (98) | Analog out -> GPIO35. Needs 24-48 h burn-in. |
| 9 | Sound sensor | Electronic Spices KY-037 - https://www.amazon.in/dp/B08PQSVDP1 | Rs 140 | 3.7* (64) | Analog out -> GPIO34. KY-037 = KY-038 with a better mic; `adc` platform works with either. |
| 10 | LD2410C (presence radar) | HLK-LD2410C 24 GHz radar - https://www.amazon.in/dp/B0BXDLHHH2 | Rs 999 | 4.9* (49) | UART -> GPIO17/18. Needs the external `ld2410` ESPHome component. |

**Phase 4 subtotal: ~Rs 2,050.**

## What NOT to buy

- **37-in-1 sensor kit** (~Rs 919): you'd use only ~3 of 37 modules (IR receiver, sound
  sensor, DHT11), and it is missing every critical part - no IR transmit LED, no MQ-135,
  no presence radar, no ESP32 board, no breadboard, no jumper wires.
- Any **Arduino Uno starter bundle**: bundles a board you don't need and 30 modules you
  won't wire.

## Review-quality caveat

Indian generic-electronics ratings run noisy (inflated 5-stars, tiny counts). Picks were
weighted by review count. The ESP32 board and the KY-037 sound sensor are the
weakest-reviewed lines - the listed alternates are the fallback if stock or ratings shift.
