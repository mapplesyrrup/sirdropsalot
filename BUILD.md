# ESP32 Weather Station — Complete Build & Setup Guide

*Detailed post-printing assembly, direct-soldered wiring (no connectors), flashing, and Home Assistant configuration. Work through it in order. Every sensor is independent, so you can skip any you don't want to build — just leave its block out of the YAML in Step 7.*

---

## How the system fits together

Before touching a soldering iron, here's the whole picture so the wiring makes sense:

- Five sensors sit outdoors: **DHT22** (temp/humidity), **BMP180** (pressure, lives in the box), **anemometer** (wind speed), **wind vane** (wind direction), **rain gauge** (rainfall).
- An **ESP32** reads them all. It runs on 3.3V logic.
- Three of the sensors are **5V Hall-effect** types (anemometer, rain gauge, wind vane). Their 5V output would damage or misread on a 3.3V pin, so each signal passes through a **level shifter** first.
- The ESP32 runs **ESPHome** firmware, joins your WiFi, and pushes readings to **Home Assistant**, where you see live values, history graphs, and dashboards.

Data path: *sensor → (level shifter if 5V) → ESP32 GPIO → ESPHome → WiFi → Home Assistant → dashboard/app.*

---

## Parts checklist

**Core electronics**
- 1× ESP32 dev board — the external-antenna version is worth it if the station will be far from your router or behind a wall.
- 1× DHT22 (AM2302) temperature/humidity sensor.
- 1× BMP180 barometric pressure sensor (I2C).
- 6× Hall-effect sensors: 4 for the vane, 1 for the anemometer, 1 for the rain gauge.
- 3× (or 3-channel) bidirectional logic level shifters — one channel per 5V Hall signal.
- 1× pull-up resistor for the DHT22 data line, 4.7kΩ–10kΩ.
- Up to 3× 1kΩ resistors as pull-ups for the pulse/Hall signal lines (see the notes — these prevent phantom counts).

**Wiring & assembly**
- Perfboard (several small pieces — one per sensor cluster plus a main board).
- Solid or stranded hookup wire; an old CAT5/CAT6 network cable is ideal for the outdoor runs (it gives you 8 color-coded conductors in one sheathed cable).
- Heat-shrink tubing in a couple of sizes, plus electrical tape.
- Solder, flux, soldering iron.

**Mechanical / mounting**
- M4 & M5 threaded rods, matching bolts and nuts, including self-locking (nylon-insert) nuts.
- 1× ball bearing for the wind vane pivot.
- Neodymium magnets (small — for the vane top and the anemometer hub).
- Rectangular aluminium profile for the frame + offcuts for legs.
- Small acrylic offcut to cap the vane sensor box.
- Weatherproof junction box for the electronics, cable glands for wire entry.
- Hot glue, drill, micro-USB cable.

> **Sensor choice tip:** If you haven't bought the Hall sensors yet, **DRV5032** parts run natively on 3.3V and draw microamps instead of milliamps. Using them lets you delete the level shifters entirely and simplifies every 5V run below to a plain 3.3V run. The rest of this guide assumes the original 5V sensors; adapt as needed.

---

## Step 1 — Temperature & humidity sensor (DHT22)

**Why the Stevenson screen matters:** direct sun heats the sensor body and you'll read temperatures several degrees too high. The louvered 3D-printed screen shades the sensor while letting outside air circulate freely across it, and it keeps rain off. Don't skip it or mount the sensor in a solid box.

**Build:**
1. Solder the DHT22 to a small perfboard. The DHT22 has 4 pins but only 3 are used: **VCC (3.3V), DATA, GND** (the third pin is usually unused/NC).
2. Solder the **pull-up resistor (4.7k–10k) between the DATA pin and the 3.3V pin.** The DHT22's data line is open-drain and needs this to read reliably; without it you get dropouts or NaN readings.
3. Solder your three-conductor run **directly** to VCC / DATA / GND — no connector. Slip heat-shrink over each joint before soldering, then shrink it. Note the wire colors you used for each; write it down.
4. Fit the perfboard inside the Stevenson screen. Stack the printed louver layers and clamp them together with the two threaded rods and nuts. Glue the sensor board so the sensor element sits in the airflow, not pressed against a wall.
5. Attach the bracket underneath to hold the whole screen up on the frame later.

**Wiring:** DATA → **GPIO 4**. VCC → 3.3V. GND → GND. (3.3V device — no level shifter.)

---

## Step 2 — Wind vane (direction)

This detects which of 8 directions the wind blows from using 4 Hall sensors and a magnet that spins with the vane.

**Mechanical:**
1. **Balance the arrow.** Push a bolt into the tail (arrow-feather) end as a counterweight against the pointed tip, so the vane doesn't droop and rotates freely.
2. **Seat the ball bearing** into the printed base so the upper piece spins with almost no friction. A stiff bearing will make the vane ignore light winds.
3. Slide the vane assembly onto its threaded rod and lock it with a **self-locking nut** so vibration doesn't loosen it.

**Sensors & magnet:**
4. Mount the **4 Hall sensors** on the base at the North, East, South, West positions (90° apart).
5. Raise the sensors **~1 cm above the base**, using the printed spacers/standoffs. This is important — the M4/M5 bolts elsewhere in the base are ferrous and will attract/interfere with the magnet if the sensors sit too low.
6. Glue the **magnet to the underside of the rotating top piece**, positioned so it passes directly over each sensor as the vane turns. This top piece doubles as a rain cap over the sensors.
7. Check magnet polarity: Hall sensors are polarity-sensitive. Spin the vane by hand and confirm each sensor triggers (its onboard LED lights) as the magnet passes. If one never triggers, flip the magnet over.

**Wiring the four sensors:**
8. Each Hall sensor board has 3 wires: **VCC (5V), GND, SIGNAL**. Solder the four boards onto a shared perfboard.
9. Run **one GND and one 5V** out to all four boards (tie them common on the perfboard), and bring **four separate signal wires** back toward the box. This is exactly what a CAT5 cable is good for: use one pair for 5V+GND and the remaining conductors for the four signals.
10. Solder all runs directly and heat-shrink them. House the four boards in the printed box with the acrylic offcut on top to keep rain out.

**Wiring (through level shifters — see Step 6):** N → **GPIO 33**, W → **GPIO 25**, S → **GPIO 26**, E → **GPIO 27**.

**How direction is derived:** each sensor reads the direction the vane currently faces. A pure cardinal (say North) fires one sensor; an intermediate direction (North-East) sits between two magnets' positions and fires **two** adjacent sensors at once. Software turns the four on/off states into one of 8 labels (Step 8).

---

## Step 3 — Anemometer (wind speed)

**Mechanical:**
1. Print the base and the rotating hub. Attach the three cups to the hub with the three screws.
2. Fit **one Hall sensor** on the fixed base and glue **one magnet** to the rotating hub, positioned to sweep past the sensor once per revolution.
3. The sensor emits **one pulse per revolution.** The ESP32 counts pulses over time to get RPM, then converts to wind speed.

**Wiring (through a level shifter):** SIGNAL → **GPIO 19**, plus 5V and GND.

### Converting pulses to wind speed — do this properly

The original guide multiplies pulses by `0.18` to get km/h, but the author openly admits this is a guess. Here are three approaches, worst to best:

**A. Quick estimate (physics of half-sphere cups)**
$$V\ (\text{km/h}) = 71.1 \times f \times r$$
where **f** = revolutions per *second* and **r** = radius from the center of rotation to the center of a cup, in meters. (Derivation: for hemispherical cups the drag-coefficient ratio gives a "cup factor" of ~3.14, and `V = 3.14 × 2π × f × r`, which reduces to `≈19.75 × f × r` in m/s, or `≈71.1 × f × r` in km/h.)

Example: if r = 0.065 m and the anemometer spins at 5 rev/s, V ≈ 71.1 × 5 × 0.065 ≈ **23 km/h**.

**B. Convert that into the ESPHome multiplier.** ESPHome's `pulse_counter` reports pulses per minute by default. 1 pulse = 1 revolution, so at r = 0.065 m:
- pulses/min → rev/s = (pulses/min ÷ 60)
- V(km/h) = 71.1 × (pulses/min ÷ 60) × 0.065 ≈ **0.077 × pulses/min**

So `multiply: 0.077` is a physics-based starting point for r = 6.5 cm — quite different from 0.18. Measure your own r and recompute.

**C. Empirical calibration (most accurate).** Aerodynamic losses mean no formula is exact. On a calm day, tape the anemometer to a car roof, and at a steady 10, 20, 30, 40 km/h (use GPS/speedometer) log the pulses/min ESPHome reports. Plot speed vs. pulses/min; the slope is your true multiplier. This automatically folds in bearing friction and cup inefficiency.

---

## Step 4 — Rain gauge (rainfall)

A tipping-bucket mechanism: rain fills a tiny seesaw bucket, it tips when full, a magnet on the seesaw passes a Hall sensor, and the sensor changes state. **Each state change = a fixed volume of water = a fixed depth of rain in mm.**

**Wiring (through a level shifter):** SIGNAL → **GPIO 23**, plus 5V and GND.

**Calibrating mm-per-tip:** the YAML uses `multiply: 0.173`, meaning each counted pulse = 0.173 mm of rain. To verify yours: slowly pour a known volume of water (e.g. from a syringe) into the funnel and count the tips. mm per tip = (poured volume in mL ÷ funnel collection area in cm²) × 10 ÷ number of tips. Adjust the multiplier to match your gauge's geometry.

> **Important reliability note (from builders who hit this):** the pulse-counting sensors — rain gauge and anemometer — can register phantom counts, and in one case the ESP32 counted pulses *even with a signal wire physically disconnected*, which points to a floating input. **Add a 1kΩ pull-up resistor on each pulse signal line** (between signal and 3.3V on the ESP32 side of the level shifter). One builder fixed erratic wind-speed counts this way and it's cheap insurance for the rain gauge too.

---

## Step 5 — Aluminium frame (optional but recommended)

Mounting all three outdoor sensors on one rigid profile makes final installation a single job.

1. Use a length of rectangular aluminium profile as the main crossbar.
2. Fix the **rain gauge in the center**, the **anemometer and wind vane on the two ends** so the cups and vane clear each other and the crossbar doesn't shadow the wind.
3. Cut two more profile pieces as **legs**. On the end of each leg, drill one large hole so the mounting bolt head sits recessed and flush — nothing protrudes underneath to rock on the mounting surface.
4. Bolt it all together. Keep the vane and anemometer level so the bearings load evenly.

Skip this if you're mounting the sensors directly to an existing structure like a roof edge or railing — just keep them clear of large obstructions that disturb airflow.

---

## Step 6 — Electronics box & wiring (direct-soldered, no connectors)

Everything terminates in a weatherproof junction box. Since we're soldering direct rather than using JST/screw terminals, plan the layout first and test before sealing.

**Assembly order:**
1. Mount the **ESP32** in the box. If you're using the external-antenna board, run the antenna out through a gland or mount it on the box underside for the best signal.
2. Install the **BMP180**. It's I2C: **SDA → GPIO 21**, **SCL → GPIO 22**, plus **3.3V** and **GND**. Drill a small vented grid on the underside of the box beneath it so it samples true outside air pressure but stays rain-shielded. BMP180 is a 3.3V part — no level shifter.
3. Bring the **DHT22** run in and solder DATA → GPIO 4 (3.3V device, direct).
4. **Level shifters for the three 5V Hall signals** (anemometer, rain gauge, vane ×4 — seven signals total, so size your shifter channel count accordingly). On each shifter:
   - **HV side:** 5V supply + the 5V sensor signal in.
   - **LV side:** 3.3V supply + the shifted signal out to the ESP32 GPIO.
   - Tie the shifter's HV↔5V and LV↔3.3V references correctly or it won't translate.
5. **Direct soldering approach (no connectors):**
   - Solder each incoming sensor run straight to its perfboard pad or the shifter pin.
   - Insulate **every** joint with heat-shrink slipped on *before* soldering.
   - For the shared rails (GND, 5V, 3.3V), make a small **perfboard "bus"**: a row of linked pads for each rail. Solder every ground to the GND bus, every 5V to the 5V bus, etc. This is far cleaner and more reliable than twisting wires together.
   - Add the **1kΩ pull-ups** on the pulse signal lines (Step 4 note) here at the ESP32 side.
   - Bundle and dress the wires with cable ties.

### Full pin map

| Sensor / signal | ESP32 GPIO | Voltage | Via level shifter? |
|---|---|---|---|
| DHT22 data (temp/humidity) | 4 | 3.3V | No |
| BMP180 SDA | 21 | 3.3V | No |
| BMP180 SCL | 22 | 3.3V | No |
| Rain gauge pulse | 23 | 5V | Yes |
| Wind speed pulse | 19 | 5V | Yes |
| Wind vane — North | 33 | 5V | Yes |
| Wind vane — West | 25 | 5V | Yes |
| Wind vane — South | 26 | 5V | Yes |
| Wind vane — East | 27 | 5V | Yes |
| Light sensor (ADC, optional) | 35 | — | No (analog) |

> **Because it's all soldered and sealed:** label every wire with tape before soldering, keep a wiring photo/diagram, and run the Step 11 tests **with the box still open**. Reworking a soldered joint inside a mounted, sealed box outdoors is genuinely miserable. The upside is durability — soldered-and-shrunk joints handle outdoor vibration and humidity far better than plug connectors.

---

## Step 7 — Flash the ESP32 with ESPHome

You do **not** write an Arduino sketch. ESPHome compiles firmware from YAML and uploads it.

1. Install **Home Assistant** on a Raspberry Pi or in a VM (many OS-specific tutorials exist).
2. In Home Assistant, open **Settings → Add-ons → Add-on Store** and install **ESPHome**.
3. Open ESPHome → **New device** → choose **ESP32**. Give it a name (e.g. `meteo`). ESPHome generates a boilerplate config with your WiFi and an API/OTA block.
4. **Edit** that device and paste the block below *after* the generated boilerplate. Put your real WiFi SSID/password in the generated `wifi:` section (not shown here).
5. Click **Install → Plug into this computer**, connect the ESP32 by micro-USB, and follow the prompts. The **first** flash must be over USB; afterward ESPHome updates it wirelessly (OTA).

```yaml
i2c:
  sda: 21
  scl: 22
  scan: true
  id: bus_a

sensor:
  - platform: dht                 # DHT22 temp + humidity on GPIO 4
    pin: 4
    temperature:
      name: "Temperatura esterna"
    humidity:
      name: "Umidità esterna"
    update_interval: 10s

  - platform: pulse_counter       # Rain gauge: counts tips on GPIO 23
    pin: 23
    count_mode:
      rising_edge: INCREMENT
      falling_edge: INCREMENT     # counts both edges = every state change
    unit_of_measurement: 'mm'
    name: 'Pioggia istantanea'
    filters:
      - multiply: 0.173           # mm of rain per counted pulse — calibrate!
    total:                        # running total that persists between updates
      unit_of_measurement: 'mm'
      name: 'Pioggia'
      accuracy_decimals: 3
      filters:
        - multiply: 0.173
    update_interval: 5s

  - platform: bmp085              # BMP180 uses the bmp085 platform (I2C)
    temperature:
      name: "Temperatura centralina"
    pressure:
      name: "Pressione esterna"
    update_interval: 10s

  - platform: pulse_counter       # Anemometer: pulses on GPIO 19
    pin: 19
    unit_of_measurement: 'Km/h'
    name: 'Velocità del vento'
    filters:
      - multiply: 0.18            # pulses/min -> km/h; replace with YOUR value
    update_interval: 5s

  - platform: uptime              # handy for spotting reboots/instability
    name: Uptime

  - platform: adc                 # optional light sensor on GPIO 35
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

**What each block does:**
- `i2c:` sets up the two-wire bus the BMP180 talks on. `scan: true` prints found I2C addresses to the log — useful for confirming the BMP180 is wired right.
- `dht` reads temperature and humidity every 10s.
- The two `pulse_counter` blocks count Hall-sensor pulses. `multiply` filters convert raw pulse counts into real units — **these are the two numbers you calibrate** (rain mm/tip and wind km/h).
- `bmp085` is the correct ESPHome platform name for the BMP180.
- `uptime` lets you catch unexpected resets on the dashboard.
- The four `binary_sensor` entries are the raw vane sensors — combined into a direction in the next step.

*Rename the Italian labels to English if you like, but if you do, update the matching entity IDs in Steps 8–10.*

---

## Step 8 — Turn 4 vane signals into one wind direction

Two ways to do this. **Option B (on-device) is cleaner** and is what I'd recommend, but Option A matches the original guide.

### Option A — Home Assistant template sensor
Edit `configuration.yaml` (install the **File Editor** or **Studio Code Server** add-on if needed) and add under `sensor:`

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

Then **Developer Tools → YAML → Check Configuration**, and if valid, **Restart Home Assistant**. Note the logic keys on `== 'off'` because these particular Hall sensors read *low/off* when the magnet is present — verify that matches your sensors' behavior (watch the states in Developer Tools while turning the vane).

### Option B — do it entirely in ESPHome (recommended)
Replace the four plain `binary_sensor` entries from Step 7 with the version below, which adds an on-device `text_sensor` that outputs the 8-point direction directly. No `configuration.yaml` edits, and all wind logic stays on the device.

```yaml
binary_sensor:
  - platform: gpio
    pin: 33
    name: "WS Wind North"
    id: wind_north
    filters:
      - invert:            # Hall sensor reads low when magnet present; invert to "true = detected"
      - delayed_on: 100ms  # debounce brief gusts/flutter
  - platform: gpio
    pin: 25
    name: "WS Wind West"
    id: wind_west
    filters:
      - invert:
      - delayed_on: 100ms
  - platform: gpio
    pin: 26
    name: "WS Wind South"
    id: wind_south
    filters:
      - invert:
      - delayed_on: 100ms
  - platform: gpio
    pin: 27
    name: "WS Wind East"
    id: wind_east
    filters:
      - invert:
      - delayed_on: 100ms

text_sensor:
  - platform: template
    name: "Wind Direction"
    lambda: |-
      if (id(wind_north).state && id(wind_east).state) { return {"NE"}; }
      else if (id(wind_north).state && id(wind_west).state) { return {"NW"}; }
      else if (id(wind_south).state && id(wind_east).state) { return {"SE"}; }
      else if (id(wind_south).state && id(wind_west).state) { return {"SW"}; }
      else if (id(wind_north).state) { return {"N"}; }
      else if (id(wind_south).state) { return {"S"}; }
      else if (id(wind_east).state) { return {"E"}; }
      else if (id(wind_west).state) { return {"W"}; }
      else { return {"Unknown"}; }
```

The `invert` filter fixes the low-when-detected behavior so `state == true` means "magnet here," and `delayed_on: 100ms` stops the direction from flickering on momentary gusts.

---

## Step 9 — Rain totals that survive reboots (utility meter)

ESPHome's rain `total` resets to zero whenever the ESP32 reboots. A Home Assistant **utility meter** gives you clean per-period totals that persist and auto-reset on a schedule.

1. **Settings → Devices & Services → Helpers → Create Helper → Utility Meter.**
2. Pick the rain entity (`sensor.pioggia`) as the source.
3. Name the counter (e.g. "Rain Today") and set the reset cycle: **Daily**.
4. Repeat to create additional counters — "Rain This Month" (Monthly), "Rain This Year" (Yearly), etc.

Now the dashboard can show today's, this month's, and total rainfall independently, and a mid-cycle reboot won't wipe them.

---

## Step 10 — Build the dashboard

1. Add live cards (Entities card or individual gauge/sensor cards) for temperature, humidity, pressure, wind speed, wind direction, and rain totals.
2. For graphs, install **mini-graph-card** via **HACS** (Home Assistant Community Store). It's great for a temperature graph with daily min/max and a pressure-trend graph — a falling pressure line is your rough "weather getting worse" indicator.
3. **Finding entity IDs:** Home Assistant derives them from the friendly name — "Temperatura esterna" becomes `sensor.temperatura_esterna` (lowercased, spaces → underscores, accents stripped). If a graph card won't find an entity, don't guess — open **Developer Tools → States** and copy the exact ID. This is the single most common dashboard snag.

Example mini-graph-card for temperature:
```yaml
type: custom:mini-graph-card
name: Temperatura
entities:
  - sensor.temperatura_esterna
hours_to_show: 24
points_per_hour: 2
show:
  extrema: true      # shows the day's min and max
```

4. **History logging:** enable/configure the **recorder** integration (and optionally a longer-term database) so graphs go back further than the default retention.
5. **Phone access:** install the Home Assistant companion app. For access away from home, **Nabu Casa Cloud** is the simplest (paid) route; self-hosted remote access (VPN, reverse proxy) is free but takes more setup and care to secure.

---

## Step 11 — Bench test, then calibrate

Do all of this **before** sealing the box and mounting outside.

**Functional tests (box open, ESP32 powered):**
- In ESPHome, open the device **Logs**. Confirm it connects to WiFi and that `i2c scan` finds the BMP180 address (usually 0x77). If not, recheck SDA/SCL.
- Confirm temperature, humidity, and pressure show sane values.
- Spin the anemometer by hand — wind speed should rise and fall. Watch for it counting when stationary (floating input → add/verify the 1k pull-up).
- Tip the rain bucket by hand a few times — the rain count should increment one step per tip, and *only* when you tip it.
- Rotate the vane slowly through a full circle and confirm the direction reads through all 8 points (N, NE, E, SE, S, SW, W, NW) in order. If two directions are swapped, you have two signal wires crossed — fix now while it's easy.

**Calibration:**
- **Anemometer:** apply your measured multiplier from Step 3 (physics estimate or, better, the car-roof calibration). Update `multiply` under the wind `pulse_counter`.
- **Rain gauge:** pour a known water volume, count tips, compute mm/tip, update both `multiply` values (instantaneous and total) under the rain block.
- **Pressure:** compare BMP180 pressure to a nearby official station; if it reads consistently off, you can add an offset filter. Remember stations usually report sea-level-adjusted pressure while the BMP180 reads absolute — expect a difference proportional to your altitude.

---

## Weatherproofing & mounting

- Use **cable glands** wherever wires enter the box; seal any drilled holes you aren't using.
- Point drilled vents (BMP180 grid, box drainage) **downward** so rain can't pool or run in.
- Add a small **drip loop** in cables entering the box so water runs off the low point instead of tracking into the enclosure.
- 3D-printed parts degrade in UV. **PLA gets brittle in sunlight within a season or two.** Prefer **ASA** or **PETG** for outdoor parts; if you already printed in PLA, paint/UV-coat the parts or plan to reprint. Print structural parts at a higher infill (~40%+) and enough walls to survive wind load.
- Mount the frame solidly and level, clear of walls and roof eddies that would distort wind readings. Higher and more open = better wind data.

---

## Troubleshooting quick reference

| Symptom | Likely cause / fix |
|---|---|
| ESP32 won't connect to WiFi after adding sensors | Weak signal — use the external-antenna board / relocate; also confirm you didn't introduce a YAML error that stops boot. Check ESPHome logs over USB. |
| DHT22 reads NaN / drops out | Missing or wrong pull-up on DATA; wire run too long; bad solder joint. Add 4.7k–10k pull-up. |
| Rain or wind counts pulses while stationary | Floating/noisy input — add 1kΩ pull-up on that signal line. |
| Wind direction two points swapped | Two vane signal wires crossed — re-check GPIO 33/25/26/27 vs N/W/S/E. |
| Vane always reads one direction or "Unknown" | Magnet polarity wrong (flip it), sensors too close to ferrous bolts (raise the ~1cm standoffs), or `invert` filter missing. |
| BMP180 missing from I2C scan | SDA/SCL swapped, no 3.3V, or bad joint. `scan: true` in `i2c:` should list its address. |
| Entity not found in dashboard card | Copy the exact entity ID from Developer Tools → States; don't type it from the friendly name. |
| Rain total resets to zero randomly | The ESP32 rebooted — that's expected; use the Step 9 utility meter for persistent totals, and check `Uptime`/logs for why it reset. |

---

You're done. Test with the box open, calibrate the two multipliers, weatherproof, mount high and clear, and point the antenna for a solid WiFi link. From there the data logs itself and the pressure trend gives you a homemade forecast.
