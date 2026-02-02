# Nerduino 🚰📟  
**Nerduino** (Greek: *Nero* = water + Arduino) is a DIY project that monitors:

- **Water level in two tanks** using **waterproof ultrasonic sensors** (JSN‑SR04T)
- **Water inlet pressure** using an **analog pressure transducer**
- **Network access** via an **Arduino Uno R3 + Ethernet Shield**

It exposes a simple web dashboard and a JSON API so you can view values in a browser or integrate them into systems like Home Assistant.

---

## What you are building (in plain words)

Imagine you have one or two water tanks (cisterns). You want to know:

1. **How full each tank is** (in liters and percent)  
2. **What pressure is on the water inlet pipe** (in bar)

Nerduino does that by:

- Measuring the **distance** from a sensor mounted above the water surface to the water surface (ultrasonic “ping” like a bat).
- Converting that distance into a **liquid height**, and then into **liters** and **percent**.
- Reading an **analog voltage** from the pressure sensor and converting it into **bar**.
- Serving the results on your local network via Ethernet.

---

## Live web endpoints

Once the Arduino is running on your network:

- `http://<device-ip>/`  
  → **Dashboard** (HTML, auto-refresh)
- `http://<device-ip>/api/status`  
  → **JSON API** (Home Assistant friendly)
- `http://<device-ip>/raw`  
  → **CSV output** for debugging

---

## Hardware / Bill of Materials

### Required parts
- **Arduino Uno R3**
- **Ethernet Shield** (W5100/W5500 compatible, using `Ethernet.h`)
- **2× JSN‑SR04T** waterproof ultrasonic sensors (HC‑SR04 compatible in “Ping Mode”)  
  Amazon (example): https://www.amazon.de/-/en/dp/B0FHWGWNK4
- **1× Analog pressure transducer (3-wire)**, 0–10 bar range (example used in code)  
  Amazon (example): https://www.amazon.de/-/en/dp/B07T5BVV1D?th=1
- Wires / terminals / enclosure / cable glands / strain relief
- A stable 5V power supply (Ethernet shields draw current spikes)

### Strongly recommended (cheap parts that massively improve stability)
If your sensors are several meters away (you mentioned **~4 m** for pressure and **~10–15 m** for ultrasonic), add these parts.  
They prevent “random readings”, glitches and noise.

#### For **each ultrasonic sensor** (Tank 1 and Tank 2)
- **1× Pull-down resistor** on ECHO: **47 kΩ – 100 kΩ** (typical: **100 kΩ**)
- **2× Series resistors**: **100 Ω – 330 Ω** (typical: **220 Ω**)  
  (one in TRIG, one in ECHO)
- **1× Decoupling capacitor**: **100 nF (0.1 µF)** between VCC and GND **near the sensor**
- Optional but helpful: **10 µF** near the sensor (helps with slower voltage dips)

#### For the **pressure sensor (A0)**
- **1× Pull-down resistor**: **100 kΩ** from A0 to GND
- **1× Series resistor**: **1 kΩ** in the signal line
- **1× Capacitor**: **100 nF** from A0 to GND (near the Arduino)

> If you only add a few parts:  
> **Pressure:** 1 kΩ + 100 nF + 100 kΩ at A0  
> **Ultrasonic:** 100 kΩ on ECHO + 100 nF near sensor power

---

## Why the extra resistors/capacitors help (beginner-friendly explanation)

Long cables are not “perfect wires”. Over **4–15 meters**, they behave like:

- **Antennas** (they pick up electrical noise from pumps, mains wiring, relays, motors)
- **Capacitors/inductors** (they distort fast digital signals)
- **Voltage drop** (especially when you power sensors over long 5V wires)

This can cause:
- digital input pins to read **random HIGH/LOW** (called “floating inputs”)
- ultrasonic pulse measurement to glitch → wrong distances
- analog readings to jump around → pressure spikes or “floating” detection

Your firmware already contains error detection states (e.g. `noise`, `timeout`, `press_floating`).  
The extra components reduce those errors and make readings stable.

### 1) Pull-down resistors (ECHO→GND, A0→GND)
**Problem:** If a sensor is unplugged or contact is bad, the Arduino pin can “float”.  
A floating pin can randomly pick up noise and look like valid data.

**Fix:** A pull-down resistor gently forces the input to a known safe state (LOW / near 0V) when the sensor isn’t actively driving the line.

- ECHO pull-down prevents fake pulses and weird distances.
- A0 pull-down prevents random ADC values if the pressure signal is disconnected.

**Why 100 kΩ?** It is strong enough to stop floating, but weak enough not to load the sensor output.

### 2) Series resistors (TRIG/ECHO and analog signal)
**Problem:** Long cables can cause overshoot/ringing (like a spring).  
Fast digital edges can bounce, creating extra transitions.

**Fix:** A small resistor in series “damps” the edge and reduces ringing.
It also adds minor protection against transients/ESD.

Typical values:
- Ultrasonic: **220 Ω** in TRIG and ECHO
- Pressure signal: **1 kΩ** (also used for the RC filter)

### 3) Decoupling capacitors near sensors (100 nF + optional 10 µF)
**Problem:** Sensors and Ethernet shields cause small current spikes. Over long wires, this can lead to tiny voltage dips at the sensor.

**Fix:** A capacitor near the sensor acts like a tiny local energy buffer and a noise shunt to ground.

- **100 nF** is good for fast high-frequency noise
- **10 µF** helps with slower dips

### 4) RC low-pass filter on pressure signal (1 kΩ + 100 nF)
**Problem:** Analog lines pick up noise → the ADC reads unstable values.

**Fix:** 1 kΩ in series + 100 nF to ground form a simple low-pass filter:
- It removes fast noise but still reacts quickly to real pressure changes.
- Your sampling interval is ~1 second, so this does not “lag” in practice.

---

## Pin mapping / Wiring (matches `main.ino`)

### Ethernet Shield reserved pins (do NOT use for sensors)
The Ethernet shield uses SPI. Avoid these pins for sensors:

- **D10** → Ethernet CS/SS (Chip Select). **Reserved**
- **D11** → SPI MOSI
- **D12** → SPI MISO
- **D13** → SPI SCK
- **D4**  → SD card CS (if shield has SD). Keep HIGH to disable SD.
- **D0/D1** → Serial USB. Avoid if you want serial debugging.

### Sensor wiring used by this project

#### Tank 1 ultrasonic (JSN‑SR04T)
- VCC  → **5V**
- GND  → **GND**
- TRIG → **D2**
- ECHO → **D3**

#### Tank 2 ultrasonic (JSN‑SR04T)
- VCC  → **5V**
- GND  → **GND**
- TRIG → **D5**
- ECHO → **D6**

#### Pressure sensor (3-wire analog)
- V+   → **5V**
- GND  → **GND**
- OUT  → **A0**

---

## Recommended “stabilized wiring” (for long cables)

### Pressure sensor (A0) – recommended circuit (place near Arduino)
Add these parts close to the Arduino (short wires on a small proto board or terminal block):

```
Sensor OUT ----[ 1kΩ ]-----+----- A0 (Arduino)
                           |
                         [100nF]
                           |
                          GND

A0 (Arduino) ----[100kΩ]---- GND
```

Also recommended near the pressure sensor (at the far end of the long cable):
- **100 nF** from **5V** to **GND** (optional + **10 µF**)

### Ultrasonic sensors – recommended additions (each sensor)
Near the Arduino:
- TRIG series resistor: **220 Ω**
- ECHO series resistor: **220 Ω**
- ECHO pull-down: **100 kΩ** to GND

Example for Tank 1:

```
Arduino D2 ----[220Ω]---- TRIG (sensor)
Arduino D3 ----[220Ω]----+---- ECHO (sensor)
                          |
                        [100kΩ]
                          |
                         GND
```

Near each sensor module:
- **100 nF** between **VCC** and **GND** (optional + **10 µF**)

---

## Using UTP cable properly (important!)

UTP (twisted pair) cable is a good choice *if you wire it correctly*.  
The biggest beginner mistake is using one shared ground for many signals over long distance.

### Golden rule
For every signal line, use a **ground wire in the same twisted pair** as the return path.

Twisting helps cancel interference and reduces noise pickup.

### Suggested pair assignment (ultrasonic sensor, per sensor)
- Pair 1: **TRIG + GND**
- Pair 2: **ECHO + GND**
- Pair 3: **5V + GND** (power)
- Pair 4: spare / extra ground / future

### Suggested pair assignment (pressure sensor)
- Pair 1: **OUT + GND**
- Pair 2: **5V + GND**
- Pair 3/4: spare

### Power over long cables
5V over 10–15 m can drop depending on wire gauge and current. If sensors become unstable:
- Use multiple wires in parallel for **5V** and **GND**, OR
- Consider sending **12V/24V** and stepping down to 5V near the device (buck converter)

---

## Software overview (what the firmware does)

### Sampling and filtering
- Sampling interval: **1 second**
- Ultrasonic robustness:
  - 3 measurements per tank
  - plausibility checks for too-small or too-large values
  - median/average selection
  - exponential moving average (EMA) smoothing when OK
- Pressure robustness:
  - reads many ADC samples (`PRESS_SAMPLES`)
  - calculates spread (min/max difference) to detect floating/noisy input
  - validates voltage range before mapping to bar

### Diagnostic states
The firmware reports explicit states so you can detect wiring issues:

Ultrasonic:
- `ok`
- `timeout` (no echo)
- `noise` (implausible tiny echo)
- `out_of_range`
- `blind` (below reliable minimum distance)
- `disconnected`

Pressure:
- `ok`
- `press_out_of_range`
- `press_floating` (ADC spread too high)

If a reading is invalid, it becomes JSON `null` (not a fake number).

---

## Configuration (edit this for your tanks)

In `main.ino`, each tank has a config:

- `diameterCm` — inner diameter of the tank (horizontal cylinder)
- `totalLiters` — total volume at 100% full
- `fullAtDCm` / `emptyAtDCm` — distance from sensor to water surface at full/empty clamp points
- `minValidCm` — sensor blind zone threshold (JSN‑SR04T is typically unreliable below ~23–25 cm)

Example:
```cpp
TankConfig tank1Cfg = {
  2, 3,              // trigPin, echoPin
  30000UL,           // echoTimeoutUs
  25.0f,             // minValidCm
  120.0f,            // diameterCm
  2200.0f,           // totalLiters
  5.0f,              // fullAtDCm
  120.0f             // emptyAtDCm
};
```

### Pressure calibration (very important)
Many 3-wire pressure sensors output **0.5–4.5 V** for 0–max bar (ratiometric).  
Some output **0–5 V**.

In the firmware:
```cpp
const float PRESS_V_MIN   = 0.5f;  // volts at 0 bar (typical)
const float PRESS_V_MAX   = 4.5f;  // volts at max bar
const float PRESS_BAR_MAX = 10.0f; // max bar of your sensor
```

If your sensor is truly 0–5 V, set:
- `PRESS_V_MIN = 0.0`
- `PRESS_V_MAX = 5.0`

---

## Build & upload (Arduino IDE)

1. Install Arduino IDE
2. Open the project folder
3. Open `main.ino`
4. Select:
   - Board: **Arduino Uno**
   - Port: your Arduino COM port
5. Upload

After upload, open Serial Monitor at **115200** baud to see the IP address.

---

## Troubleshooting

### Ultrasonic readings jump or show `noise/timeout`
- Ensure ECHO pull-down (100 kΩ) is installed
- Add 220 Ω series resistors in TRIG and ECHO
- Add 100 nF decoupling at the sensor (VCC/GND)
- Use twisted pairs correctly: TRIG+GND and ECHO+GND
- Avoid mounting above turbulence/foam/water inlet stream

### Pressure shows `press_floating` or values jump
- Add RC filter at A0: 1 kΩ + 100 nF
- Add 100 kΩ pull-down at A0
- Pair OUT with GND in the same twisted pair
- Add decoupling at the sensor power

### Ethernet unreliable
- Use a stable power supply (Ethernet shield can draw spikes)
- Keep D10 reserved for Ethernet CS
- Keep D4 HIGH to disable SD if present (code already does this)

---

## Safety notes
- This project is for monitoring. Do not use it for safety-critical control.
- Water + electricity: use enclosures, cable glands, drip loops.
- Pressure plumbing: correct fittings and ratings; never exceed sensor range.

---

## License
Apache License 2.0

This project is licensed under the Apache License, Version 2.0.  
You may obtain a copy of the License at:
http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under
the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
either express or implied. See the License for the specific language governing permissions
and limitations under the License.
