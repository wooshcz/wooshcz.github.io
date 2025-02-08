---
title: "LaskaKit E-Paper Kit With Home Assistant"
date: 2025-02-05T20:56:48+01:00
draft: false
---

This is just a note to future self on how I made the LaskaKit E-Paper Kit work with Home Assistant and ESPHome.

Just a bit of context: The [kit](https://www.laskakit.cz/laskakit-live-7-5-e-paper-stavebnice-s-wifi-pro---zivy-obraz-/?variantId=13529) I got my hands on contains a E-Paper display and a custom ESP32-based board called ESPink which also comes pre-flashed with a custom firmware. You can build a firmware for this board yourself and the code is available in [GitHub](https://github.com/LaskaKit/ESPink). However I wasn't interested in running the board with this custom FW as I wanted to connect it with my local Home Assistant instance.

The Kit comes with different display options, I got the 7.5 inch 800x480 screen from GoodDisplay with part number [GDEY075T7](https://www.good-display.com/product/396.html). It supports four levels of grey, but I am currently using it in BW mode only, I'll get to that later.

## ESPHome Configuration

I flashed ESPHome onto the ESP32 board and since I already had some prior experience with a older and smaller E-Paper screen which I integrated using `waveshare_epaper` display component, I wanted to give this a go. The [Waveshare E-Paper Component page](https://esphome.io/components/display/waveshare_epaper.html) does not explicitly mention this display as supported, but after a bit of googling I've realized that it should actually work under one of the many `7.50in` models.

My ESPHome config for the ESP32 board starts with the `esphome` and `esp32` sections:

```yaml
esphome:
  name: esphome-web-0055c8
  friendly_name: esp32-ESPink
  min_version: 2024.11.0
  name_add_mac_suffix: false
  on_boot: 
    then:
      - switch.turn_on: epaper_power

esp32:
  board: esp32dev
  flash_size: 16MB
  framework:
    type: esp-idf
```

I'll skip the other unrelated configuration options which are pretty much standard and so on and go straight to the most interesting bit, which is the configuration for the display component itself:

```yaml
spi:
  clk_pin: GPIO18
  mosi_pin: GPIO23

display:
  - platform: waveshare_epaper
    id: disp01
    cs_pin: GPIO5
    dc_pin: GPIO17
    busy_pin:
      number: GPIO4
      inverted: true
    reset_pin: GPIO16
    update_interval: 300s
    rotation: 0
    model: 7.50inv2
```

I found the relevant pin numbers for this board in the product listing of ESPink on the [LaskaKit shop](https://www.laskakit.cz/laskakit-espink-esp32-e-paper-pcb-antenna/?variantId=12419). Another piece to the puzzle was to be found in the Waveshare E-Paper Component page where in a big red warning box you can find:

> The BUSY pin on the `gdew0154m09` and **Waveshare 7.50in V2 models must be inverted** to prevent permanent display damage. Set the pin to `inverted: true` in the config.

With the `busy_pin` now set as `inverted` the rest of the setup is pretty straightforward and you can just use the official Home Assistant and ESPHome documentation and you should be fine. The display refreshes once every 5 minutes and it takes around 3 seconds to complete.

## Known Issues

- The display currently does work only in black and white mode, but the hardware should support 4 levels of grey.
- The display does support partial refresh, but the `7.50inv2` model parameter I'm currently using can only do full refresh. There is however a [Pull Request](https://github.com/esphome/esphome/pull/7751) open that looks like could address this -- it'll be available as `7.50inv2p` in the list of already existing 7.5 inch implementations and models that the components supports.

## Final Result

![LaskaKit connected showing data from Home Assistant](/images/laskakit-with-homeassistant.jpg)
