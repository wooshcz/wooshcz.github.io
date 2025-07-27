---
title: "Departure Board on Led Matrix Display"
date: 2025-07-24T21:56:44+02:00
draft: false
---

Recently I saw a cool project someone shared on the ESPHome Discord server that inspired me and I've decided that I'll attempted to replicate it and build something similar. It was a LED matrix display based departure board connected to an Arduino that was displaying publicly available information about upcoming departures of the nearest public transport stop.

Since I live in Prague where we happen to have a publicly available data where you can fetch live information about location of trams/buses/metro and their delays I thought it should be pretty much doable. The public transport APIs are available at [Golemio API](https://api.golemio.cz/docs/openapi/index.htm), you must register to get your API Key and then you're good to go. The default rate-limit policies are pretty generous and completely sufficient for this home project.

## Hardware

- 2x Waveshare RGB LED matrix display 64x32 with 2.5mm pitch. I've bought these over [rpishop.cz](https://rpishop.cz/maticove-displeje/5985-waveshare-rgb-plnobarevny-led-maticovy-panel-6432-pxl-roztec-25-mm.html). There is some documentation along with some examples [on the Waveshare Wiki](https://www.waveshare.com/wiki/RGB-Matrix-P2.5-64x32).
- ESP32 development board. I've this over [rpishop.cz](https://rpishop.cz/esp32-a-esp8266/1500-esp32-vyvojova-deska.html) as well, but I suspect you can use almost any ESP32 board you might be having laying around.
- 5V/4A power supply with a 5mm barrel plug for the LCD Display is recommended. I'm using one which is rated for 3A and it is able to provide enough current for both of the panels but I suspect it's because I run it at half brightness and also most of the pixels are off.
- 5V power supply with micro USB plug for the ESP32.

The Waveshare LED display comes with all necessary wires plus also contains the cable which allows you to chain multiple of these displays together. In the end I've chained two of them so in the code you can program it like it would be one 128x32 large panel.

## Home Assistant Code

I've decided to use Home Assistant's [RESTful integration](https://www.home-assistant.io/integrations/rest) to communicate with the Golemio API endpoint and process the raw departure data for three distinct bus stops that happen to be near me. I'm using the [`/v2/pid/departureboards` endpoint](https://api.golemio.cz/pid/docs/openapi/index.htm#/%F0%9F%9A%8F%20PID%20Departure%20Boards%20(v2)/get_v2_pid_departureboards) and the following `rest.yaml` platform config:

```yaml
- scan_interval: 30
  resource: "https://api.golemio.cz/v2/pid/departureboards?ids[]=Uxxx&ids[]=Uyyy&ids[]=Uzzz&minutesBefore=0&minutesAfter=360&includeMetroTrains=false&airCondition=true&skip=canceled&limit=6"
  headers:
    X-Access-Token: !secret golemio_api_key
  sensor:
    - name: Bus Departures Raw
      value_template: "OK"
      json_attributes:
        - departures
```

This fetches the live positions and the predicted departure times for the 6 upcoming bus lines (including a real delay of each bus) that happen to stop at any of those 3 stop IDs (set in the query string in the URL resource). It extracts all objects from the `departures` array from the JSON string and stores it to the sensor attribute. Originally I've had three different sensors for three different bus stops but I've opted for this approach for multiple reasons but mostly because it reduces the level of duplication in HA.

The raw JSON response looks like this:

```json
{
    "stops": [
        { ... }
    ],
    "departures": [
        {
            "arrival_timestamp": { ... },
            "delay": { ... },
            "departure_timestamp": { ... },
            "last_stop": { ... },
            "route": { ... },
            "stop": { ... },
            "trip": { ... }
        },
        { ... }
    ],
    "infotexts": []
}
```

Second piece to the puzzle is to parse this information and pick only the data I'm interested in displaying on the LCD: `route.short_name`, `trip.headsign`, `delay.seconds` and `departure_timestamp.predicted`. I want to display the 3 upcoming departures only (since you can't reasonably fix more text to the display without pagination), so I've created a template sensor which parses the raw sensor data stored in the seteps above and save the nearest 3 departures -- each in one attribute to simplify the parsing later in the ESPHome code:

```yaml
  - name: "Upcoming Bus Departures"
    state: "OK"
    attributes:
      line1: >
        {% set buses = state_attr('sensor.bus_departures_raw', 'departures') %}
        {% if buses %}
          {%- set bus = buses[0] %}
            {%- set route = bus['route']['short_name'] %}
            {%- set sign = bus['trip']['headsign'] or "?" %}
            {%- set delay = bus['delay']['seconds'] | default(0) %}
            {%- set minutes = (
                  (as_timestamp(bus['departure_timestamp']['predicted']) - as_timestamp(now()))
                  / 60) | round(0, 'floor') %}
            {%- set line = "{}|{}|{}|{}".format(route, sign, delay, minutes) %}
            {{ line }}
        {% endif %}
      line2: >
        ...
          {%- set bus = buses[1] %}
        ...
      line3: >
        ...
          {%- set bus = buses[2] %}
        ...
```

This results in a sensor having three attributes (`line1`, `line2` and `line3`) where each is having those 4 pieces of information separated by a pipe:

```text
line1: 120|Na Knížecí|0|17
line2: 120|Nádraží Radotín|124|19
line3: 104|Na Hvězdárně|7|35
```

Now we can expose this sensor to ESPHome and try to display this text on our display.

## ESPHome Configuration

The display is actually not supported out of the box in ESPHome and I've had to use [this external component](https://github.com/TillFleisch/ESPHome-HUB75-MatrixDisplayWrapper). By far the biggest chunk of the code is the lambda for the `display` component, but after I figured out all the X and Y positions for my elements on the screen the rest was fairly straightforward. What actually ended up being the biggest challenge was picking the best font! Even with this pitch there is really only about ~10 pixels for each row and some fonts are really unusable when rendered in this size. I've ended up using Doto Bold.

```yaml
esphome:
  name: esp32-nodemcu
  friendly_name: esp32-nodemcu

esp32:
  board: esp32dev
  framework:
    type: arduino

external_components:
  - source: github://TillFleisch/ESPHome-HUB75-MatrixDisplayWrapper@main

font:
  - file: "gfonts://Doto@Bold"
    id: mono
    size: 10
    glyphsets:
      - GF_Latin_Core
  
display:
  - platform: hub75_matrix_display
    id: matrix
    width: 64
    height: 32
    brightness: 64
    chain_length: 2
    G2_pin: GPIO33
    C_pin: GPIO18
    OE_pin: GPIO32
    pages:
      - id: page1
        lambda: |-
          int destPosX = 22;
          int delayPosX = 95;
          int timePosX = 127;

          auto red = Color(255, 0, 0);
          auto yellow = Color(255, 0, 255);
          auto orange = Color(255, 30, 150);
          auto green = Color(0, 50, 255);
          auto white = Color(255, 255, 255);
          auto grey = Color(150, 150, 150);

          std::string line1 = id(deps_line1).state;
          std::string line2 = id(deps_line2).state;
          std::string line3 = id(deps_line3).state;

          std::array<std::string, 3> lines = { line1, line2, line3 };

          for (int i = 0; i < lines.size(); ++i) {
            int y = i * 10;
            const auto &line = lines[i];

            std::string line_number, head_sign;
            float delay;
            int minutes;

            size_t start = 0;
            size_t end = line.find('|');
            if (end != std::string::npos) {
                line_number = line.substr(start, end - start);
                start = end + 1;
            }

            end = line.find('|', start);
            if (end != std::string::npos) {
                head_sign = line.substr(start, end - start);
                start = end + 1;
            }

            end = line.find('|', start);
            if (end != std::string::npos) {
                delay = parse_number<int>(line.substr(start, end - start)).value() / 60.0;
                start = end + 1;
            }

            minutes = parse_number<int>(line.substr(start)).value(); // last part

            Color color = green;  // default fallback
            if (delay >= 9) {
              color = red;
            } else if (delay >= 6) {
              color = orange;
            } else if (delay >= 3) {
              color = yellow;
            }

            it.printf(1, y, id(mono), white, COLOR_OFF, TextAlign::TOP_LEFT, "%s", line_number.c_str());
            it.printf(destPosX, y, id(mono), grey, COLOR_OFF, TextAlign::TOP_LEFT, "%s", head_sign.substr(0, 16).c_str());
            it.printf(timePosX, y, id(mono), color, COLOR_OFF, TextAlign::TOP_RIGHT, "%d", minutes);
          }

number:
  - platform: hub75_matrix_display
    matrix_id: matrix
    id: matrix_brightness
    name: "Display Brightness"

switch:
  - platform: hub75_matrix_display
    matrix_id: matrix
    name: "Display Power"
    restore_mode: ALWAYS_ON
    id: power

text_sensor:
  - platform: homeassistant
    id: deps_line1
    entity_id: sensor.upcoming_bus_departures
    attribute: line1
  - platform: homeassistant
    id: deps_line2
    entity_id: sensor.upcoming_bus_departures
    attribute: line2
  - platform: homeassistant
    id: deps_line3
    entity_id: sensor.upcoming_bus_departures
    attribute: line3
```

## Final Result

As you can see in the picture it doesn't look too bad. However I've decided to trim the text of the final destination for each bus because for certain names it would start to overlap with the last number - the minutes until departure. This is a bit hacky and I'll think of improving this later.

![Matrix Display showing the upcoming departures](/images/departure-board-matrix-display.jpg)
