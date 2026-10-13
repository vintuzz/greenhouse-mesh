# Smart Greenhouse IoT Mesh Network

![ESP8266](https://img.shields.io/badge/Hardware-Wemos_D1_Mini_(ESP8266)-blue)
![Raspberry Pi](https://img.shields.io/badge/Broker-Raspberry_Pi-C51A4A)
![MQTT](https://img.shields.io/badge/Protocol-MQTT_(Mosquitto)-660066)
![Node-RED](https://img.shields.io/badge/Automation-Node--RED-8F0000)
![C++](https://img.shields.io/badge/Firmware-C%2B%2B_%2F_Arduino-00599C)

An autonomous, decoupled Internet of Things (IoT) mesh network designed to monitor environmental parameters, automate irrigation, and ensure operator safety inside a smart greenhouse. 

Developed at **Politechnika Lubelska (Lublin University of Technology)** during the Erasmus+ exchange program (Course: *Internet of Things [E060]*, Academic Year 2025/2026) within a 12-student international engineering team.

📄 **[Read the Full Technical Report (PDF)](./report.pdf)**

---

## 🎯 Role & Key Contributions

Within the 12-member working group, my work focused on system integration, central automation logic, and embedded actuator development:

1. **Communication Architecture & Standardization:** Defined the shared MQTT topic hierarchy and designed the universal JSON dataframe standard adopted by all 12 team members to ensure seamless interoperability across heterogeneous nodes.
2. **Node-RED Co-Design (Automation Engine):** Co-designed the six core Node-RED flows running on the central Raspberry Pi coordinator, implementing state caching, hysteresis-based irrigation, security overrides, and occupational safety alerts.
3. **Relay Node Firmware & Hardware Deployment:** Designed and programmed the C++ firmware for the **Wemos D1 Mini (ESP8266)** actuator node controlling the $5\text{ V}$ relay water pump, featuring NTP epoch synchronization, stack-allocated JSON parsing, and resilient broker reconnection.

---

## 🏗️ System Architecture

The system relies on a star/mesh publish-subscribe topology centered around a **Raspberry Pi** hosting **Eclipse Mosquitto** (MQTT Broker) and **Node-RED**. Sensors and actuators are completely decoupled and communicate strictly via MQTT over Wi-Fi.

```text
[Environmental Sensors] ──┐
[Security Sensors]      ──┼──► (MQTT Publish) ──► [Raspberry Pi: Mosquitto + Node-RED]
[Wearable BPM Sensor]   ──┘                                      │
                                                        (MQTT Subscribe)
                                                                 ▼
                                            ┌────────────────────┼────────────────────┐
                                            ▼                    ▼                    ▼
                                   [Wemos D1 Mini]        [OLED Displays]      [SD Card Logger]
                                 (Water Pump Relay)     (HMI & Emergency)      (Data Persistence)
```

### MQTT Topic Namespace

| MQTT Topic | Direction | Description | QoS |
| :--- | :--- | :--- | :---: |
| `greenhouse/sensor/soil` | Sensor $\rightarrow$ Broker | Soil moisture readings ($\%$) | $2$ |
| `greenhouse/sensor/light` | Sensor $\rightarrow$ Broker | Ambient light intensity ($\text{lux}$) | $2$ |
| `greenhouse/sensor/barometric` | Sensor $\rightarrow$ Broker | Barometric pressure & temperature | $2$ |
| `greenhouse/sensor/temp_and_humidity` | Sensor $\rightarrow$ Broker | DHT air temperature & relative humidity | $2$ |
| `greenhouse/sensor/button1` | Sensor $\rightarrow$ Broker | Physical trigger for on-demand OLED summary | $2$ |
| `greenhouse/sensor/heartrate` | Sensor $\rightarrow$ Broker | Operator heart-rate monitor ($\text{BPM}$) | $2$ |
| `greenhouse/security/motion` | Sensor $\rightarrow$ Broker | PIR motion detection events | $2$ |
| `greenhouse/security/door` | Sensor $\rightarrow$ Broker | Greenhouse door open/close state | $2$ |
| `greenhouse/actuator/relay` | Broker $\rightarrow$ Relay | Command topic for the water pump (`ON` / `OFF`) | $0$ |
| `greenhouse/actuator/relay/status` | Relay $\rightarrow$ Broker | Execution confirmation & telemetry feedback | $0$ |
| `greenhouse/actuator/oled1` | Broker $\rightarrow$ OLED 1 | On-demand environmental summary display | $0$ |
| `greenhouse/actuator/oled2` | Broker $\rightarrow$ OLED 2 | Critical security & medical emergency alerts | $0$ |
| `sd_logger/save` | Broker $\rightarrow$ Logger | Formatted log strings for SD card persistence | $0$ |

---

## 📡 Shared JSON Dataframe Specification

To guarantee forward compatibility and smooth integration among nodes developed independently, all devices adhere to a strict JSON schema:

**Inbound Command (Broker $\rightarrow$ Relay Node):**
```json
{
  "action": "ON"
}
```

**Standardized Outbound Telemetry / Feedback (Node $\rightarrow$ Broker):**
```json
{
  "device_id": "cuzzocrea",
  "device_type": "actuator",
  "timestamp": 1716800000,
  "action": "ON",
  "data": {}
}
```

---

## ⚙️ Embedded Firmware: Wemos D1 Mini Relay Node

The irrigation actuator node is built on an **ESP8266 (Wemos D1 Mini)** wired to a single-channel $5\text{ V}$ relay switching a water pump on digital pin `D1`.

### Engineering Highlights
* **Fail-Safe Boot Initialization:** Drives `D1` to `LOW` immediately upon boot before initializing network stacks, preventing accidental pump activation during power cycles.
* **Zero Heap Fragmentation:** Uses `StaticJsonDocument<256>` from `ArduinoJson` to allocate memory strictly on the stack—crucial for long-term stability on an ESP8266 with only $80\text{ KB}$ of usable RAM.
* **Strict Payload Validation:** Validates incoming MQTT payloads against deserialization errors before triggering hardware GPIO changes, ignoring malformed packets or unexpected metadata fields.
* **NTP Time Synchronization:** Integrates `NTPClient` over UDP (`pool.ntp.org`) to attach accurate Unix epoch timestamps to every status feedback packet for distributed log correlation.
* **Resilient Broker Reconnection:** Implements a non-blocking retry mechanism that generates a randomized hexadecimal Client ID (`WemosClient-XXXX`) on each attempt, preventing broker session lockouts after sudden Wi-Fi drops.

### Dependencies
* `ESP8266WiFi`
* `PubSubClient` (by Nick O'Leary)
* `ArduinoJson` (v6 by Benoit Blanchon)
* `NTPClient` & `WiFiUdp`

---

## 🧠 Node-RED Automation Engine

The central coordinator logic on the Raspberry Pi is modularized across **six distinct Node-RED flow tabs** to enforce separation of concerns:

1. **Global Context & Datalogging:** Subscribes to `greenhouse/sensor/#`, caching the latest environmental readings in RAM context variables to avoid redundant network traffic, while simultaneously formatting and forwarding logs to `sd_logger/save`.
2. **Hysteresis Irrigation Logic:** Prevents hardware wear and plant thermal shock through dual-condition evaluation:
   * **Pump Activation (`ON`):** Triggered only when soil moisture drops to a critical threshold ($< 30\%$) **AND** sunlight is indirect ($< 2000\text{ lux}$), preventing leaf scorch during peak daylight.
   * **Pump Deactivation (`OFF`):** Triggered once optimal soil saturation is reached ($> 80\%$).
3. **Security & Operator Override:** Interfaces with the PIR motion detector and door sensor. If human presence is detected while irrigation is active, the flow immediately dispatches an `OFF` override command to avoid spraying operators and triggers a visual warning.
4. **On-Demand HMI (`OLED 1`):** Listens to `greenhouse/sensor/button1` to fetch cached environmental metrics from RAM and render an instant summary without polling sensors continuously.
5. **Occupational Health Monitoring (`OLED 2`):** Analyzes real-time telemetry from the operator's wearable heart-rate sensor. If bradycardia ($< 40\text{ BPM}$) or tachycardia ($> 130\text{ BPM}$) is detected, an immediate `"MEDICAL EMERGENCY"` alert is broadcast to the emergency display.

---

## 📂 Suggested Repository Structure

```text
├── docs/
│   └── report.pdf                 # Full technical report and code analysis
├── firmware/
│   └── relay_node_esp8266.ino     # Wemos D1 Mini C++ source code
├── node-red/
│   └── flows.json                 # Exported Node-RED automation flows
└── README.md                      # Project documentation
```

---

## 👤 Author

**Vincenzo Cuzzocrea**  
*Computer Engineering Student @ University of Catania | Full-Stack & IoT Developer*  
* GitHub: [@vintuzz](https://github.com/vintuzz)
* Email: vincuzzocrea@gmail.com