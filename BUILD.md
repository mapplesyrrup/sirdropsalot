# ESP32 Weather Station — Build & Setup Guide

*Post-printing assembly, wiring, and software. Follow in order; you can skip any sensor you don't want to build.*

---

## What you'll need on the bench

**Electronics**
- ESP32 board (external WiFi antenna version recommended)
- DHT22 temperature/humidity sensor + one pull-up resistor (~4.7k–10k)
- BMP180 pressure sensor (I2C)
- 6× Hall effect sensors (4 for the wind vane, 1 for the anemometer, 1 for the rain gauge)
- 5V→3.3V logic level shifters (for the 5V Hall sensors)
- Perfboard, JST connectors, an old network cable for sensor runs
- Micro-USB cable, junction/plastic box for the electronics

**Hardware**
- M4 & M5 threaded rods, bolts, nuts (incl. self-locking nuts)
- Ball bearing for the wind vane, neodymium magnets
- Aluminium profile for the frame, acrylic offcut for the vane sensor cover
- Hot glue, solder, drill

> **Tip from the comments:** If you're still buying parts, consider **DRV5032 Hall sensors** instead of A311x. They run on 3.3V (so you can skip the level shifters) and draw microamps instead of milliamps.

---

## Step 1 — Assemble the temperature sensor (DHT22)

1. Solder the DHT22 onto a small perfboard with a connector.
2. Add a pull-up resistor between the **3.3V** line and the **signal** pin.
3. Mount the perfboard inside the 3D-printed **Stevenson screen** (the louvered enclosure that shades it from sun/rain while letting air through — this is what keeps temperature readings accurate).
4. Join the printed layers with two threaded rods, glue the sensor board inside, and fit the bracket underneath to hold it up.

Signal wire → **GPIO 4**.

---

## Step 2 — Assemble the wind vane

1. Balance the arrow: bolt as counterweight at the tail, tip at the front.
2. Seat the ball bearing in the printed base so the top spins freely.
3. Mount **4 Hall sensors** on the base at N / E / S / W, raised ~1 cm off the base to keep them away from metal bolts that could trigger the magnet.
4. Glue the magnet to the rotating top piece (this piece also caps the sensors against rain).
5. Slide the vane onto the threaded rod, secure with a self-locking nut.
6. Wire the 4 sensor boards on a perfboard. Run GND and 5V to them (network cable works well) and bring the 4 signal lines back to the ESP32.

Signal wires → **GPIO 33 (N), 25 (W), 26 (S), 27 (E)**.

**Logic:** each sensor reads the cardinal point the vane faces; intermediate directions (e.g. NE) fire two sensors at once. You'll convert these four on/off signals into a readable direction in software (Step 8).

---

## Step 3 — Assemble the anemometer

1. Print base + rotating hub; attach the three cups to the hub with screws.
2. Fit one Hall sensor on the base and a magnet on the rotating part — one pulse per revolution.

Signal wire → **GPIO 19**.

**Wind-speed conversion:** the guide uses a rough `rpm × 0.18` for km/h, but it's admittedly approximate. Better options from the comments:
- Physics-based (half-sphere cups): `V (km/h) = 71.1 × f × r`, where `f` = revolutions/second and `r` = radius from center to cup center in meters.
- Most reliable: **calibrate empirically** — mount it on a car roof on a calm day and log pulses against the speedometer at 5/10/15/20 mph.

---

## Step 4 — Assemble the rain gauge

Uses a tipping mechanism with a Hall sensor: each state change = a fixed amount of rain (measured in mm of height). Mount it and route the signal.

Signal wire → **GPIO 23**.

> **Known issue from the comments:** the pulse-counting sensors (rain + wind speed) can misbehave. A **1k pull-up resistor on the signal line** fixed intermittent/false counting for at least one builder — worth adding preemptively.

---

## Step 5 — Build the frame (optional)

Fix all three outdoor sensors to a single rectangular aluminium profile: rain gauge in the center, anemometer and wind vane on the sides. Cut two more profile pieces as legs (drill a large hole on one end so the mounting bolt sits flush). Skip this if you already have a mounting surface like a roof edge.

---

## Step 6 — Wire the electronics box

1. Mount the ESP32 in the junction box; attach the external WiFi antenna underneath.
2. Add the **BMP180** pressure sensor. It's I2C — connect **SDA → GPIO 21**, **SCL → GPIO 22**, plus 3.3V and GND. Drill a small vent grid under it so it reads real air.
3. Bring the DHT22 signal straight to the ESP32 (3.3V device, no shifter).
4. For the **5V Hall sensors** (anemometer, rain gauge, wind vane), route each signal through a **level shifter** to drop 5V → 3.3V. Solder the shifters to perfboard.
5. Use **JST connectors** between boards — sturdier than jumpers. Bundle the shared GND / 5V / 3.3V rails and tidy with cable ties.

### Pin reference

| Signal | GPIO |
|---|---|
| DHT22 (temp/humidity) | 4 |
| Rain gauge pulse | 23 |
| Wind speed pulse | 19 |
| BMP180 SDA / SCL | 21 / 22 |
| Wind N / W / S / E | 33 / 25 / 26 / 27 |
| Light sensor (ADC, optional) | 35 |

---

## Step 7 — Flash the ESP32 via ESPHome

1. Install **Home Assistant** (Raspberry Pi or a VM — plenty of tutorials online).
2. From the add-on store, install **ESPHome**.
3. In ESPHome, add a new device → select **ESP32**.
4. Paste the config below *after* the boilerplate ESPHome generates, and fill in your WiFi SSID/password where prompted.
5. Click **Install → Plug into this computer**, connect the ESP32 by USB, and follow the prompts to flash. After the first flash, future updates go over WiFi (OTA).

> You do **not** need to write a separate Arduino sketch — ESPHome compiles and uploads everything from this YAML.

```yaml
i2c:
  sda: 21
  scl: 22
  scan: true
  id: bus_a

sensor:
  - platform: dht
    pin: 4
    temperature:
      name: "Temperatura esterna"
    humidity:
      name: "Umidità esterna"
    update_interval: 10s

  - platform: pulse_counter
    pin: 23
    count_mode:
      rising_edge: INCREMENT
      falling_edge: INCREMENT
    unit_of_measurement: 'mm'
    name: 'Pioggia istantanea'
    filters:
      - multiply: 0.173
    total:
      unit_of_measurement: 'mm'
      name: 'Pioggia'
      accuracy_decimals: 3
      filters:
        - multiply: 0.173
    update_interval: 5s

  - platform: bmp085
    temperature:
      name: "Temperatura centralina"
    pressure:
      name: "Pressione esterna"
    update_interval: 10s

  - platform: pulse_counter
    pin: 19
    unit_of_measurement: 'Km/h'
    name: 'Velocità del vento'
    filters:
      - multiply: 0.18
    update_interval: 5s

  - platform: uptime
    name: Uptime

  - platform: adc
    pin: 35
    name: "Luminosità esterna"
    unit_of_measurement: "%"
    update_interval: 10s
    filters:
      - multiply: 30

binary_sensor:
  - platform: gpio
    pin: 33
    name: "Vento direzione NORD"
  - platform: gpio
    pin: 25
    name: "Vento direzione OVEST"
  - platform: gpio
    pin: 26
    name: "Vento direzione SUD"
  - platform: gpio
    pin: 27
    name: "Vento direzione EST"
```

*Rename the Italian labels to English if you prefer — just keep the entity references consistent in the next steps. Adjust the rain `multiply` (0.173) and wind `multiply` (0.18) once you've calibrated your own hardware.*

---

## Step 8 — Turn the 4 wind sensors into a readable direction

The vane reports four separate on/off states. Convert them to one direction with a template sensor. Edit **`configuration.yaml`** (install the File Editor add-on if you haven't) and paste under `sensor:`

```yaml
- platform: template
  sensors:
    direzione_vento:
      friendly_name: Direzione del vento
      value_template: >-
        {% if states('binary_sensor.vento_direzione_ovest') == 'off' and states('binary_sensor.vento_direzione_nord') == 'off' %}
        NORD-OVEST
        {% elif states('binary_sensor.vento_direzione_est') == 'off' and states('binary_sensor.vento_direzione_nord') == 'off' %}
        NORD-EST
        {% elif states('binary_sensor.vento_direzione_ovest') == 'off' and states('binary_sensor.vento_direzione_sud') == 'off' %}
        SUD-OVEST
        {% elif states('binary_sensor.vento_direzione_est') == 'off' and states('binary_sensor.vento_direzione_sud') == 'off' %}
        SUD-EST
        {% elif states('binary_sensor.vento_direzione_nord') == 'off' %}
        NORD
        {% elif states('binary_sensor.vento_direzione_est') == 'off' %}
        EST
        {% elif states('binary_sensor.vento_direzione_sud') == 'off' %}
        SUD
        {% elif states('binary_sensor.vento_direzione_ovest') == 'off' %}
        OVEST
        {% endif %}
```

Then **Developer Tools → Check Configuration**, and if valid, **restart Home Assistant**.

> **Cleaner alternative (from the comments):** do this entirely in ESPHome with a `text_sensor` + lambda instead of `configuration.yaml`. Add `filters: [invert:, delayed_on: 100ms]` to each wind `binary_sensor` — `invert` fixes the fact that the Hall sensor reads *low* when it detects the magnet, and `delayed_on` debounces brief gusts. That keeps all the wind logic on the device.

---

## Step 9 — Configure the rain totals (utility meter)

ESPHome sends a running rain total that resets to zero if the ESP32 reboots. Use a Home Assistant **utility meter** to get clean per-period totals that survive reboots:

1. **Settings → Devices & Services → Helpers → Create Helper → Utility Meter.**
2. Select the rain gauge entity and name the counter.
3. Set the reset cycle (e.g. daily, monthly).
4. Repeat to create separate counters for each interval you want to display (today's rain, monthly rain, etc.).

---

## Step 10 — Build the dashboard

1. Add real-time cards for each sensor (temperature, humidity, pressure, wind speed, wind direction, rain).
2. For graphs, install the **mini-graph-card** from HACS (Home Assistant Community Store) — good for temperature min/max and the pressure trend you'll use for rough forecasting.
3. Entity names follow the pattern `sensor.<friendly_name_lowercased_with_underscores>` — e.g. "Temperatura esterna" → `sensor.temperatura_esterna`. If an entity won't appear, check its exact ID under **Developer Tools → States**.
4. To log history, enable/configure the **recorder** integration and point it at a database.
5. For phone access, install the Home Assistant app. For access outside your home network, **Nabu Casa Cloud** is the easiest (paid) option.

---

## Step 11 — Test & calibrate

- Watch the LEDs on the Hall sensor boards — they confirm the sensors trigger correctly.
- Spin the vane through all 8 directions and confirm the template sensor updates.
- If rain or wind-speed counts look wrong or count with a wire disconnected, add/verify the pull-up resistors (Step 4).
- Calibrate the anemometer multiplier once you have a reference (see Step 3), then update the `multiply` value in the YAML.
- Check the rain `multiply` against a known volume of water poured through the gauge.

You're done — mount it, point the antenna for good WiFi, and watch the data roll in.
