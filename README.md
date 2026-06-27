# PowerDisplayESPHome

### UPDATE: Updated to handle the change to 15 min pricing by NordPool. 
**Note:** Code has been updated to support ESPHome 2026.6.2. It hasn't been tested on older versions.


This is a small display that shows the current electricity consumption, together with a graph of the today's electricity price, using either NordPool or Tibber. The software pulls the data from a Home Assistant instance, so all sources must be available there.

This is a port of the previous repo [PowerDisplayHomeAssistant](https://github.com/johannyren/PowerDisplayHomeAssistant) to ESPHome. The ESPHome version allows for much more flexibility and ease of setup than the previous version, and gets its time directly from Home Assistant rather than ntp. This simplifies handling of Daylight Savings Time etc. It also allows for more fonts, as it uses Google fonts that supports non-English characters, and can be easily replaced as desired.

**Note:** The ESPHome version requires an ESP32 microcontroller, as the ESPHome display library requires more memory than is available on a Wemos D1 mini.

## Changes in this fork

This fork adapts the original project:

- **Cheap Yellow Display (CYD).** It is configured for the all-in-one [ESP32-2432S028R "Cheap Yellow Display"](https://github.com/witnessmenow/ESP32-Cheap-Yellow-Display) board, which combines an ESP32 and a 2.8" SPI display (ST7789V driver) on a single PCB — the panel is wired on-board, so no separate display wiring is needed. The display uses ESPHome's `mipi_spi` platform (the older `ili9xxx` platform was removed in ESPHome 2026.4.0).
- **English & Euro.** All on-screen text is in English, and prices/costs are shown in €/kWh and €. The price-level thresholds live in `power_display.h`.
- **Bosch BME680 air-quality sensor.** An optional BME680 is read over I²C via the Bosch BSEC library, exposing temperature, humidity, pressure, IAQ (indoor air quality), CO₂-equivalent and VOC to Home Assistant.

Tested on ESPHome 2026.6.2.

![alt text](https://github.com/johannyren/PowerDisplayESPHome/blob/main/images/Display1.jpg?raw=true)

## Hardware

The hardware is the **ESP32-2432S028R "Cheap Yellow Display"** — an ESP32 with an on-board 2.8" ST7789V SPI display and backlight. The display pins are fixed on the board; for reference this fork uses:

```
Display (on-board SPI)      BME680 (I²C, address 0x77)
SCK   -> GPIO14             SDA -> GPIO27
MOSI  -> GPIO13             SCL -> GPIO22
MISO  -> GPIO12             VCC -> 3.3V
CS    -> GPIO15             GND -> GND
D/C   -> GPIO2
LED   -> GPIO21 (backlight)
```

The BME680 is optional — connect it to the CYD's exposed I²C pins, or remove the `bme680_bsec` block and the related sensors from `power-display-esphome.yaml` if you don't use it.

<details>
<summary>Original discrete ESP32 + ILI9341 wiring (the design this was forked from)</summary>

![alt text](https://github.com/johannyren/PowerDisplayESPHome/blob/main/images/Wiring_ILI9341.jpg?raw=true)

![alt text](https://github.com/johannyren/PowerDisplayESPHome/blob/main/images/Wiring_ESP32.jpg?raw=true)

</details>

## Files required

Copy the following files into your Home Assistant ESPHome folder, for example /homeassistant/config/esphome:

```
power_display.h
electrical_tower32.png
solar_energy32.png
```
Use the file power-display-esphome.yaml to create your ESPHome entity.

## Datasources and connection to Home Assistant
Data sources from Home Assistant are defined in power-display-esphome.yaml

It will require the HomeAssistant integration with NordPool (https://github.com/custom-components/nordpool), as well as a device that can read the current power usage from the power meter. 

### Popular ESPHome implementations are:

- Home Assistant Glow pulse counter. (https://klaasnicolaas.github.io/home-assistant-glow/)
- ESPHome HAN port reader  (https://github.com/psvanstrom/esphome-p1reader)

The implementation also expects a Utility Meter entity in Home Assistant. The following is an example to put in configuration.yaml:
```
 utility_meter:
 produktion_huset_per_dag:
   source: sensor.cumulative_active_export
   cycle: daily
```

## Backlight entity
PowerDisplayESPHome creates an entity in Home Assistant that can be used to control the brightness of the display, or turn it off with a schedule etc.

![alt text](https://github.com/johannyren/PowerDisplayESPHome/blob/main/images/Backlight_entity.jpg?raw=true)



## Casing
STL files are available for 3D printing a casing for the display.

    
