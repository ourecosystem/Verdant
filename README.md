# 🌿 Verdant — ESP32 EnviroMonitor

A real-time environmental monitoring dashboard for ESP32-based sensor setups. Tracks oxygen, CO₂, soil moisture, and light levels from your device, with a live MJPEG camera feed from an ESP32-CAM module — all in a single self-contained HTML file.

---

## Features

- **O₂ monitoring** — animated breathing ring with colour-coded alerts (normal / elevated / low)
- **CO₂ monitoring** — live ppm readout with Normal / Elevated / Danger thresholds
- **Soil moisture** — percentage gauge with descriptive status labels
- **Light level** — lux gauge with auto-scaling (lx / k lx)
- **30-reading sparkline history** for all four sensors
- **ESP32-CAM live feed** — MJPEG stream with snapshot and fullscreen support
- **Event log** — timestamped connection and sensor activity log
- **Demo mode** — generates realistic data automatically when no device is connected
- **AI-powered feedback loop** — in-app feedback form with Claude-generated responses
- **Zero dependencies** — single `.html` file, no build step, no framework

---

## How It Works

Open the file in your browser and you'll land straight on the live dashboard. No login, no install, no internet connection required — it runs entirely in the browser and talks directly to your ESP32 over your local network.

When no device is connected, the dashboard runs in **demo mode** automatically, cycling through realistic sensor values so you can explore the interface before your hardware is ready. The moment you connect a real ESP32, live data takes over.

---

## What You'll See on the Page

The dashboard is laid out in a dark green instrument-panel style, divided into several areas:

**Header bar** — across the top. On the left is the Verdant logo. On the right you'll find the ESP32 IP address input, a Connect button, a live/disconnected status indicator, and the Feedback button.

**O₂ card** (top-left) — a large animated ring gauge showing the current oxygen percentage. The ring glows green at normal levels, shifts to orange if elevated, and red if low. It pulses gently like a breath.

**CO₂ card** (top-centre) — the current CO₂ reading in parts per million, with a colour-coded badge (Normal / Elevated / Danger) and a bar chart comparing the live reading against the normal baseline (400 ppm) and the danger threshold (5,000 ppm).

**Soil Moisture card** (top-right) — a horizontal gauge bar showing moisture as a percentage, with a plain-language label: Dry, Adequate, or Saturated.

**Light Level card** (second row, left) — same style gauge for ambient light in lux, auto-formatted to `k lx` for large values. Labels range from Dark through Indoor / Overcast to Full Sun.

**Summary card** (second row, centre) — a quick at-a-glance list of all four current readings with a timestamp of the last successful update.

**Config card** (second row, right) — lets you change the poll rate (2 s / 5 s / 10 s / 30 s) and the sensor endpoint path without editing any code.

**Trend charts** (bottom, spanning three columns) — four sparkline graphs showing the last 30 readings for each sensor, with a glowing line and a live value label. Useful for spotting drift or sudden changes.

**Camera panel** (right column, full height) — paste your ESP32-CAM stream URL and click Load to see the live video feed. You can take a snapshot (saved as a PNG) or go fullscreen. Below the feed is a scrolling event log showing connection activity, sensor reads, and any errors.

**Feedback button** — click it any time to open the feedback modal. Rate your experience, pick topic tags, write a comment, and submit. An AI-generated reply appears before the form closes.

---

## Getting Started

### 1. Open the dashboard

Just open `esp32-dashboard.html` in your browser — no server required.

### 2. Flash your ESP32

Your ESP32 firmware needs to expose a JSON endpoint. A minimal Arduino sketch example:

```cpp
#include <WiFi.h>
#include <WebServer.h>

const char* ssid     = "YOUR_SSID";
const char* password = "YOUR_PASSWORD";

WebServer server(80);

void handleSensors() {
  // DHT22 — requires DHT sensor library by Adafruit
  float temperature = dht.readTemperature();   // °C
  float humidity    = dht.readHumidity();      // % RH

  // MQ-135 — analog read, convert to ppm using MQ135 library or calibration curve
  int   airQuality  = mq135.getPPM();          // CO₂ ppm equivalent

  // Capacitive Soil Moisture V2.0 — map raw ADC to 0–100% (calibrate for your soil)
  int   moisture    = map(analogRead(MOISTURE_PIN), 4095, 1500, 0, 100);

  // LDR — raw ADC value 0–1023 (higher = brighter with voltage divider)
  int   light       = analogRead(LDR_PIN);

  // Thermistor — convert via Steinhart-Hart equation
  float tempTherm   = readThermistor();        // your conversion function

  String json = "{";
  json += "\"o2\":"       + String(o2, 1)  + ",";
  json += "\"co2\":"      + String(co2)    + ",";
  json += "\"moisture\":" + String(moisture) + ",";
  json += "\"light\":"    + String(light);
  json += "}";

  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.send(200, "application/json", json);
}

void setup() {
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  server.on("/sensors", handleSensors);
  server.begin();
}

void loop() {
  server.handleClient();
}
```

### 3. Connect the dashboard

1. Enter your ESP32's local IP address in the **ESP32 IP** field (e.g. `192.168.1.100`)
2. Press **Connect** — the dashboard will start polling every 5 seconds
3. Adjust the poll interval or endpoint path in the **Config** card if needed

---

## Sensor JSON Format

Your ESP32 endpoint (default: `GET /sensors`) must return:

```json
{
  "o2":        20.94,
  "co2":       412,
  "tvoc":      0.142,
  "eco2":      520,
  "hcho":      0.031,
  "dhtTemp":   23.4,
  "dhtHum":    58.2,
  "soil":      45,
  "light":     620,
  "pressure":  1013.2,
  "bmpTemp":   23.1,
  "ky28Temp":  22.9
}
```

| Field | Unit | Sensor | Description |
|-------|------|--------|-------------|
| `o2` | % vol | SC4-O2 | Oxygen concentration |
| `co2` | ppm | MH-Z1911A | True CO₂ (NDIR infrared) |
| `tvoc` | mg/m³ | Y01 | Total VOC (0–2.000) |
| `eco2` | ppm | Y01 | Equivalent CO₂ (350–2000) |
| `hcho` | mg/m³ | Y01 | Formaldehyde (0–1.000) |
| `dhtTemp` | °C | DHT22 | Air temperature (±0.5°C) |
| `dhtHum` | % RH | DHT22 | Relative humidity (±2–5%) |
| `soil` | % | Cap. Soil V2.0 | Soil moisture (0 = dry, 100 = saturated) |
| `light` | 0–1023 raw | KY-018 LDR | Ambient light ADC value |
| `pressure` | hPa | BMP085 | Barometric pressure |
| `bmpTemp` | °C | BMP085 | Temperature (I²C) |
| `ky28Temp` | °C | KY-028 | Temperature (analog GPIO 33) |

---

## ESP32-CAM Setup

The camera panel streams MJPEG video directly from your ESP32-CAM module.

1. Flash the **CameraWebServer** example sketch (included in Arduino ESP32 board support)
2. Note your module's IP address from the serial monitor
3. In the dashboard, paste the stream URL into the CAM field:
   ```
   http://<camera-ip>:81/stream
   ```
4. Press **Load ▶**

> **Tip:** The sensor ESP32 and camera ESP32-CAM can be separate devices on the same network.

---

## Your Sensors

| Sensor | Measures | Interface |
|--------|----------|-----------|
| SC4-O2 | Oxygen % concentration | Analog / UART |
| MH-Z1911A | True CO₂ (NDIR infrared) | UART (TX pin 2, RX pin 3) or PWM |
| Y01 TVOC/Gas Module (YYS) | TVOC (0–2 mg/m³), eCO₂ (350–2000 ppm), HCHO (0–1 mg/m³) | UART 9600 8N1 (4-pin JST-XH) |
| DHT22 (AM2302) | Temperature (±0.5°C) & humidity (±2–5% RH) | Digital single-wire (Adafruit DHT library) |
| KY-018 Photoresistor | Ambient light (raw ADC 0–1023) | Analog (S pin = signal) |
| Capacitive Soil Moisture V2.0 | Soil moisture % | Analog ADC → GPIO 34 |
| BMP085 | Barometric pressure & temperature | I²C (SCL + SDA) |
| KY-028 Digital Temperature | Temperature (analog voltage) | Analog → GPIO 33 |
| ESP32-CAM AI Thinker | Live video feed | MJPEG HTTP stream (:81/stream) |

---

## Configuration

All config lives in the **Config** card inside the dashboard — no code edits needed.

| Setting         | Default     | Description                          |
|-----------------|-------------|--------------------------------------|
| ESP32 IP        | 192.168.1.100 | IP address of your ESP32           |
| Poll interval   | 5 s         | How often to fetch sensor data       |
| Endpoint path   | `/sensors`  | HTTP path on your ESP32 web server   |
| CAM stream URL  | _(empty)_   | Full URL to MJPEG stream             |

---

## Feedback

The dashboard includes a built-in feedback form (click **Feedback** in the header). Submissions are processed by Claude (claude-sonnet-4-6) which generates a personalised response in real time.

Feedback categories: Sensor accuracy · Camera feed · UI/design · Charts & history · Connection issues · Feature request · Bug report

---

## Browser Support

Works in any modern browser with `fetch` and `<canvas>` support:

- Chrome / Edge 88+
- Firefox 85+
- Safari 14+

> **CORS:** Your ESP32 web server must include the `Access-Control-Allow-Origin: *` header so the browser can fetch sensor data from a different origin. See the example sketch above.

---

## Project Structure

```
esp32-dashboard.html   ← entire app (HTML + CSS + JS, single file)
README.md              ← this file
```

---

## License

MIT — free to use, modify, and deploy.
