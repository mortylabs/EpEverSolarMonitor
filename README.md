# EpEver Solar Monitor (ESP8266 + RS485 + MQTT + Home Assistant)

![EPEVER Tracer 4215BN MPPT](https://m.media-amazon.com/images/I/71k1-M5mZ9L._AC_SL1500_.jpg)

> 🔗 [Buy it on Amazon](https://amzn.eu/d/51auldm)

Monitor your off-grid solar system with style — this Arduino-based ESP8266 project connects your **EPEVER Tracer MPPT Solar Charge Controller** to **Home Assistant** over WiFi and MQTT using **RS485 Modbus**. You’ll get live voltage, current, power, temperature, and SOC readings in your dashboard — no cloud required!

---

## 🚀 Features

* Reads Modbus registers from EPEVER via RS485 (MAX485 chip)
* Publishes all metrics to MQTT as JSON
* Supports Home Assistant MQTT Discovery (with proper device\_class tagging!)
* OTA (Over-the-Air) firmware updates
* Deep sleep compatible for low-power deployments

---

## 🔌 RS485 Wiring Diagram

You can repurpose an Ethernet cable to cleanly connect your EPEVER charge controller to the RS485 breakout and NodeMCU. Here's the mapping based on colour-coded strands:

### ✳️ Ethernet Cable Colour Mapping

```
Ethernet cable Green/White → RS485MAX B
Ethernet cable Blue/White  → RS485MAX A
Ethernet cable Brown/White → NodeMCU GND
Ethernet cable Orange      → NodeMCU VIN
```

### 🔌 Component Connections

```
EPEVER RS485 ↔ RS485MAX ↔ NodeMCU 1.0 ESP8266

EPEVER COM         RS485MAX               ESP8266
-----------        --------               --------
A (+)    ───────→  A
B (−)    ───────→  B
GND      ───────→  GND      ←──────┐      GND
                                  └────── Ethernet Brown/White
                                   
RS485MAX RO ─────→ RX (GPIO3)     // RS485 Receive Out
RS485MAX DI ←──── TX (GPIO1)     // RS485 Data In
RS485MAX DE ←──── D2 (GPIO4)     // Driver Enable
RS485MAX RE ←──── D1 (GPIO5)     // Receiver Enable
RS485MAX VCC ←──── 3.3V          // Powered by NodeMCU
RS485MAX GND ←──── GND
```

> 💡 Tip: This setup allows a compact, noise-resistant tether between the controller and your ESP8266. Use stranded Cat5/Cat6 for flexibility.

![Wiring Diagram](sandbox:/mnt/data/A_README_file_displays_content_for_the_EpEver_Sola.png)

```text
EPEVER RS485 ↔ MAX485 ↔ ESP8266 (e.g. NodeMCU 1.0)

 EPEVER            MAX485            ESP8266
 --------          --------          --------
 A (+)   ─────────> RO
 B (−)   ─────────> DI
 GND     ─────────> GND              GND
                ↘  RE ←──────────── D1
                ↘  DE ←──────────── D2
 VCC     ←──────── VCC              3.3V
                               RX ←── RO
                               TX ───→ DI
```

---

## 📦 Required Libraries

Install via **Library Manager**:

* `ModbusMaster` by Doc Walker
* `PubSubClient` by Nick O’Leary
* `ESP8266WiFi` (built-in)
* `ArduinoOTA` (built-in)
* `SoftwareSerial`

---

## 🔧 Setup Instructions

1. Clone this repo and open `EpEverSolarMonitor.ino`
2. Copy `secrets_template.h` → `secrets.h` and fill in:

```cpp
const char* hostname    = "espSolarChargerOTA";
const char* mqtt_user   = "your_user";
const char* mqtt_pass   = "your_pass";
const char* mqtt_server = "192.168.1.x";
const int   mqtt_port   = 1883;
```

3. Flash to your ESP8266 via USB
4. Boot → connect to WiFi → auto-discovers in Home Assistant

---

## 📈 Published Metrics

* `pv_volt`, `pv_amps`, `pv_power`
* `batt_volt`, `batt_amps`, `batt_power`, `batt_charge`, `batt_soc`, `batt_temp`
* `load_volt`, `load_amps`, `load_power`

All published as a single JSON payload to:

```
mqtt topic: epever/state
```

---

## 📬 MQTT Discovery (Home Assistant)

Sensors auto-register under `homeassistant/sensor/epever_*`.
Each includes:

* `device_class`
* `unit_of_measurement`
* `value_template`
* `unique_id`

No YAML required.

---

## 💡 Tip

Test unsupported registers (like net battery current or energy counters) before relying on them — not all EPEVER models return all values.

---

## 📸 Screenshot (Example HA Dashboard)

![HA Screenshot](https://user-images.githubusercontent.com/placeholder.png)

---

MIT License • Made with ☀️ by MortyLabs

---

```
#define ENABLE_DISCOVERY
```

Add this to enable HA MQTT discovery — or disable it if running without Home Assistant.

## References
https://github.com/glitterkitty/EpEverSolarMonitor/blob/master/EpEverSolarMonitor.ino
https://www.instructables.com/id/Wireless-Solar-Charge-Controller-Monitor-RS485-to-/
https://github.com/tekk/Tracer-RS485-Modbus-Blynk-V2/blob/master/doc/1733_modbus_protocol.pdf
https://www.eevblog.com/forum/projects/nodemcu-esp8266-rs485-epever-solar-monitor-diy/msg2504679/#msg2504679
https://github.com/tekk/Tracer-RS485-Modbus-Blynk-V2/blob/master/Tracer-RS485-Modbus-Blynk-V2.ino
http://www.cooking-hacks.com/documentation/tutorials/modbus-module-shield-tutorial-for-arduino-raspberry-pi-intel-galileo/index.html
http://play.fallows.ca/wp/projects/electronics-projects/understanding-nodemcu-esp8266-12e-limitations/
https://4-20ma.io/ModbusMaster/group__constant.html
