# Project 3: Cloud-Connected Security Node (IoT Telemetry)

**Decode Labs — IoT Department Internship Program**
**Author:** Noor Fatima

---

## Overview

An IoT security node that broadcasts physical security telemetry across the internet to a centralized cloud dashboard. An ultrasonic distance sensor monitors a protected zone; the microcontroller joins a local Wi-Fi network and publishes live readings to a cloud MQTT broker (Adafruit IO), where they're rendered on a real-time dashboard — allowing remote monitoring from anywhere.

## Goal

Build a telemetry node that:
- Connects the microcontroller to a local **Wi-Fi network**
- Uses a lightweight protocol (**MQTT**) to publish sensor data from an ultrasonic distance sensor
- Displays the live data stream on a free IoT cloud dashboard (**Adafruit IO**)
- Continuously repeats the sense → publish → display cycle

## Key Skills

| Skill | Description |
|---|---|
| Wi-Fi Stack Configuration | Initializing the onboard radio, joining a local network, managing connection state |
| MQTT / HTTP Telemetry Protocols | Publishing structured sensor data to a remote broker over a lightweight publish/subscribe protocol |
| Cloud Dashboard Integration | Binding device data feeds to a live, remotely-accessible dashboard |

## Hardware Components

- ESP32 Dev Board (Wi-Fi + MQTT capable microcontroller)
- HC-SR04 Ultrasonic Distance Sensor
- Status LED + 220Ω resistor (local alert indicator)
- Breadboard + jumper wires
- Wi-Fi router (local network access)
- Adafruit IO account (cloud MQTT broker + dashboard)

## Wiring Summary

| Signal | From | To |
|---|---|---|
| Power (3V3) | ESP32 3V3 | Breadboard (+) rail |
| Ground | ESP32 GND | Breadboard (–) rail |
| Sensor Power | Breadboard (+) / (–) | HC-SR04 VCC / GND |
| Trigger Signal | ESP32 GPIO17 | HC-SR04 TRIG |
| Echo Signal | ESP32 GPIO16 | HC-SR04 ECHO |
| Alert Signal | ESP32 GPIO18 | LED Anode (via 220Ω resistor) |
| Wireless Link | ESP32 Wi-Fi radio | Router → Internet → Adafruit IO |

## How It Works

1. **Connect:** The ESP32 joins the local Wi-Fi network (`WiFi.begin()`) and opens a persistent MQTT connection to Adafruit IO on port 1883.
2. **Sense:** The sketch pulses the HC-SR04's TRIG pin, then times the ECHO pin's response with `pulseIn()` to calculate distance in cm.
3. **Evaluate:** If the distance falls below an intruder threshold, an alert flag is set.
4. **Publish:** The distance value (and alert flag) are published to two separate Adafruit IO feeds via MQTT.
5. **Display:** Adafruit IO renders each feed as a live chart/indicator on the dashboard, updating within a second or two of each publish.
6. **Alert Locally:** The onboard LED lights up immediately whenever the intruder threshold is crossed, independent of the cloud connection.

## ESP32 Sketch

```cpp
#include <WiFi.h>
#include "Adafruit_MQTT.h"
#include "Adafruit_MQTT_Client.h"

const char* WLAN_SSID = "YOUR_WIFI_SSID";
const char* WLAN_PASS = "YOUR_WIFI_PASSWORD";

#define AIO_SERVER      "io.adafruit.com"
#define AIO_SERVERPORT  1883
#define AIO_USERNAME    "YOUR_AIO_USERNAME"
#define AIO_KEY         "YOUR_AIO_KEY"

WiFiClient client;
Adafruit_MQTT_Client mqtt(&client, AIO_SERVER, AIO_SERVERPORT, AIO_USERNAME, AIO_KEY);
Adafruit_MQTT_Publish distanceFeed = Adafruit_MQTT_Publish(&mqtt, AIO_USERNAME "/feeds/security-distance");
Adafruit_MQTT_Publish alertFeed    = Adafruit_MQTT_Publish(&mqtt, AIO_USERNAME "/feeds/security-alert");

const int TRIG_PIN = 17;
const int ECHO_PIN = 16;
const int LED_PIN  = 18;
const float INTRUDER_THRESHOLD_CM = 30.0;
const unsigned long PUBLISH_INTERVAL_MS = 2000;
unsigned long lastPublish = 0;

void connectWiFi() {
  WiFi.begin(WLAN_SSID, WLAN_PASS);
  while (WiFi.status() != WL_CONNECTED) delay(500);
}

void connectMQTT() {
  int8_t ret;
  while ((ret = mqtt.connect()) != 0) {
    mqtt.disconnect();
    delay(3000);
  }
}

float readDistanceCM() {
  digitalWrite(TRIG_PIN, LOW);  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH); delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  return duration * 0.0343 / 2.0;
}

void setup() {
  Serial.begin(115200);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  connectWiFi();
  connectMQTT();
}

void loop() {
  if (!mqtt.connected()) connectMQTT();
  mqtt.processPackets(10);
  mqtt.ping();

  if (millis() - lastPublish >= PUBLISH_INTERVAL_MS) {
    lastPublish = millis();
    float distance = readDistanceCM();
    bool intruderDetected = (distance < INTRUDER_THRESHOLD_CM);

    distanceFeed.publish(distance);
    digitalWrite(LED_PIN, intruderDetected ? HIGH : LOW);
    alertFeed.publish(intruderDetected ? 1 : 0);
  }
}
```

## Dashboard Setup Notes (Adafruit IO)

- Create two feeds: `security-distance` and `security-alert`.
- Add a **line chart** block bound to `security-distance` for the live reading.
- Add a **status/indicator** block bound to `security-alert` to flag intrusion events.

## Testing Notes

- In **Tinkercad**, the HC-SR04's simulated distance can be adjusted to test both the "clear" and "intruder-detected" branches of the threshold logic.
- Tinkercad's simulator has **no real internet access**, so the Wi-Fi/MQTT portion must be validated on physical ESP32 hardware connected to a real access point and Adafruit IO account.
- Use the Serial Monitor to confirm Wi-Fi association, MQTT connection, and each published value before checking the dashboard.

## Suggested Improvements

- Add MQTT reconnect backoff and a watchdog timer so the node recovers automatically after a Wi-Fi drop.
- Add a moving-average filter on distance readings to reduce false alerts from sensor noise.

---
*Prepared as part of the Decode Labs IoT Internship Program.*
