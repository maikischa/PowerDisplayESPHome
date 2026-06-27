# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An ESPHome firmware project for an ESP32 + ILI9341/ST7789V SPI display that shows current household power usage and a graph of today's hourly electricity price (NordPool/Tibber). All data comes from a Home Assistant instance over the ESPHome native API — the device renders, HA supplies the values. There is no application backend; the "code" is YAML config plus a C++ header compiled into the ESPHome firmware.

## Build / flash / run

There is no test suite and no CI. The build tool is ESPHome itself, normally invoked from the Home Assistant ESPHome add-on (Install/Upload). From a CLI:

```bash
esphome compile power-display-esphome.yaml   # compile only
esphome run power-display-esphome.yaml       # compile + flash (USB first time, OTA after)
esphome logs power-display-esphome.yaml      # stream device logs
```

To deploy, the `.h` header and the PNG icons must sit next to the YAML in the HA ESPHome folder (e.g. `/homeassistant/config/esphome`), because `includes:` and `image:` reference them by path (the main YAML uses a `PowerDisplayESPHome/` subfolder prefix; the IDF YAML uses bare filenames). `!secret` keys (`encryption_key`, `ota_password`, `wifi_ssid`, `wifi_password`, `fallback_password`) come from ESPHome's `secrets.yaml`, which is not in this repo.

## Two parallel variants — keep them in sync deliberately

This repo ships two independent copies of the firmware. They share design but have diverged; a change to one is **not** automatically valid for the other.

- **Root** (`power-display-esphome.yaml` + `power_display.h`) — the maintained version. Arduino framework, single display page, ST7789V panel, includes a BME680 air-quality sensor (I²C) and 15-minute NordPool pricing (96 quarter-hour slots).
- **`esp-idf-version-with-two-displaypages/`** (`*-idf.yaml` + `power_display_idf.h`) — esp-idf framework, two display pages cycled every 5s (page 2 shows tomorrow's prices), 24-hour hourly pricing. No BME680. Has extra class methods the root lacks (`WriteTomorrowText`, `SetGraphScaleTomorrow`, `DrawPriceGraphTomorrow`, `SetAccumulatedCostToday`, etc.).

When asked to "fix the firmware," confirm which variant. Pin numbers, panel model, price-array sizing (96 vs 24), and the framework type all differ between them.

## Architecture

The whole render pipeline lives in the `display:` lambda in the YAML, which calls into the `PowerDisplay` C++ class defined in the `.h` file. Each display update (`update_interval: 10s`):

1. The lambda reads HA-backed sensor/text_sensor states (`id(...).state`) and pushes them into the class via `Set*`/`Write*` setters.
2. The class stores them in **file-scope globals** (`currentPower`, `currentPrice`, `todayMaxPrice`, `dailyEnergy`, `TodaysPrices`) — deliberately global, not class members, so they can be persisted.
3. The lambda then calls the draw methods (`DisplayIcons`, `WritePowerText`, `CreateGraph`, `SetGraphScale`, `SetGraphGrid`, `DrawPriceGraph`, etc.) which paint onto the ESPHome `display::Display` buffer.

Key cross-cutting details:

- **NVM persistence.** Globals are written to ESP32 NVS (`Preferences` library, partition `"my_partition"`) every 60s and `on_shutdown`, then re-loaded lazily inside the draw/Set methods when a value reads as `0`/`NaN`. This is what lets the display show meaningful values immediately after a reboot before HA reconnects. `SaveValuesToNVM()` is called from the YAML `interval:` and `on_shutdown:` blocks. The `Preferences` (and `Wire`) libraries must be listed under `esphome: libraries:`.
- **Price string parsing.** `SetTodaysPrices`/`SetTomorrowsPrices` receive the NordPool `today`/`tomorrow` attribute as a raw string like `[0.12, 0.15, ...]`. `SetPrices()` strips brackets and splits on space/comma into `priceArray[]` / `priceArrayTomorrow[]`. Array sizes are tied to the slot count — **96 in the root variant (15-min), 24/25 in the IDF variant (hourly)**. Changing the pricing resolution means changing the loop bounds in `DrawPriceGraph`, the array dimensions, and `SetGraphScale`'s x-axis max together.
- **Price → color/label thresholds.** The `#define`s at the top of the `.h` (`VERY_CHEAP`, `CHEAP`, `NORMAL`, ... in €/kWh) drive both the graph line color (`PriceColour`) and the text label (`WritePriceText`). Edit these to retune price bands.
- **Graph coordinate system.** `CreateGraph` sets origin/size; `SetGraphScale` derives `xFactor`/`yFactor` from `todayMaxPrice` and the slot count; all draw methods map data→pixels through those factors. Geometry constants (the x/y offsets passed in the lambda) are hand-tuned to the panel resolution and rotation.

## Conventions worth knowing

- Power sign convention: `import_el - export_el`. Positive = drawing from grid (shows tower icon), negative = exporting/solar (shows solar icon). Meters reporting kW instead of W need a `*1000` in the lambda (noted inline).
- Fonts are pulled live from Google Fonts (`gfonts://`) at compile time; the `large_text` font has an explicit `glyphs:` list including Nordic characters `å ä ö` — extend that list if you add text using new characters.
- Logging: `ESP_LOGD(...)` calls exist but most are commented out; the `logger:` is set to `ERROR` for the `component` tag to keep the bus quiet.
