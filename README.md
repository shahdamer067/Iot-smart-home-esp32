# Smart Home Automation System (ESP32 + MQTT)

A smart home prototype built with **two ESP32 boards** that talk to each other and to a phone dashboard through a **HiveMQ Cloud MQTT broker (TLS, port 8883)**. It covers lighting, fire detection, intrusion detection, temperature-based cooling and multi-method door access.

Team project for the IoT course at the Faculty of Computers and Data Science, Alexandria University (5 students).

## Features

| Subsystem | What it does |
|-----------|--------------|
| **Smart light** | Manual ON/OFF from the cloud, or automatic mode using an LDR (turns on when the room is dark). Manual control overrides auto mode. |
| **Fire alarm** | Flame and smoke sensors trigger a flashing LED and pulsing buzzer, and publish an alert. It stays on until reset from the dashboard. |
| **Security alarm** | When "secure home" is enabled, an ultrasonic sensor triggers the alarm if an object comes closer than 50 cm. |
| **Smart fan** | Turns on automatically above 35 °C, or is controlled manually (ON/OFF and LOW / MEDIUM / HIGH speed) from the cloud. |
| **Smart door** | Unlocks by RFID card, 4-digit keypad passcode, or a remote unlock command. A servo locks and unlocks the door, and the state is reported to the cloud. |

## Architecture

```
            +------------------+            +----------------------+
 Keypad,    |  ESP32 #1        |            |  ESP32 #2            |  Flame, smoke,
 RFID, IR,  |  door + fan      |            |  sensors + alarms    |  ultrasonic, LDR,
 servo,     |  (FreeRTOS tasks)|            |  (FreeRTOS tasks)    |  DHT11, LCD, LEDs,
 L298N fan  +--------+---------+            +----------+-----------+  buzzer
                     \                                /
                      \        WiFi + MQTT over TLS  /
                       +------> HiveMQ Cloud <------+
                                     |
                              Phone MQTT dashboard
```

- Each subsystem runs as its own **FreeRTOS task**, so sensors are monitored at the same time.
- ESP2 reads the temperature (DHT11) and publishes it, and ESP1 uses it for the automatic fan.
- ESP1 publishes the door state, and ESP2 mirrors it on a status LED.

## Hardware

**ESP32 #1: door and fan** (`esp1_door_fan`)
MFRC522 RFID reader (SS 21, RST 22), 4x3 keypad (rows 12, 14, 27, 26; columns 25, 33, 32), IR presence sensor (15), servo (13), door LED (34), DC fan through an L298N driver (IN1 2, IN2 4, ENA 5 for PWM).

**ESP32 #2: sensors and alarms** (`esp2_sensors_alarms`)
Flame sensor (4), smoke sensor (13), fire LED (14), ultrasonic sensor (trig 5, echo 18), security LED (27), buzzer (19), DHT11 (15), light LED (12), LDR (34), door-state LED (2), I2C LCD for WiFi/MQTT status.

## MQTT topics

| Topic | Published by | Used for |
|-------|--------------|----------|
| `front_door` | ESP1 | `LOCKED` / `UNLOCKED` |
| `unlock_button` | Dashboard | Remote unlock |
| `fan_power` | Dashboard | Fan ON/OFF |
| `fan_speed` | Dashboard | `LOW` / `MEDIUM` / `HIGH` |
| `fan_state` | ESP1 | Fan `ON` / `OFF` |
| `dht` | ESP2 | Temperature reading |
| `fire_alarm` | ESP2 | `ON` / `OFF` |
| `security_alarm` | ESP2 | `ON` / `OFF` |
| `secure_home` | Dashboard | Arm / disarm the security system |
| `online_reset_button` | Dashboard | Reset both alarms |
| `light_switch` | Dashboard | Manual light control |
| `auto_light` | Dashboard | Enable / disable automatic light |
| `light_state` | ESP2 | Light `ON` / `OFF` |

## How to run

1. Install the **Arduino IDE** and add ESP32 board support.
2. Install these libraries from the Library Manager: `PubSubClient`, `ESP32Servo`, `MFRC522`, `Keypad`, `LiquidCrystal_I2C`, `DHT sensor library` (with `Adafruit Unified Sensor`).
3. Create a free cluster at [HiveMQ Cloud](https://www.hivemq.com/mqtt-cloud-broker/) and create credentials under **Access Management**.
4. In **both** sketches, replace the placeholders at the top with your own values:
   ```cpp
   const char* ssid = "YOUR_WIFI_NAME";
   const char* password = "YOUR_WIFI_PASSWORD";
   const char* mqtt_server = "YOUR-CLUSTER-ID.s1.eu.hivemq.cloud";
   const char* mqtt_username = "YOUR_HIVEMQ_USERNAME";
   const char* mqtt_password = "YOUR_HIVEMQ_PASSWORD";
   ```
5. Wire the boards as listed above, upload `esp1_door_fan` to the first ESP32 and `esp2_sensors_alarms` to the second.
6. Control the system from any MQTT dashboard app connected to the same broker, using the topics above.

The RFID card ID and the keypad passcode in `esp1_door_fan.ino` are demo values. Change them for your own card.

## Simulation

`simulation/smart_home_packet_tracer.pkz` is a Cisco Packet Tracer model of the same system (door, fan, fire, security and light subsystems connected to a home gateway), used to test the logic before building the hardware.

## Security note

WiFi and broker credentials are **not** included in this repository. Never commit real passwords. Use your own credentials locally and keep them out of Git.

## Tech stack

ESP32, Arduino C/C++, FreeRTOS, MQTT (PubSubClient), HiveMQ Cloud, MFRC522 RFID, DHT11, L298N, Cisco Packet Tracer
