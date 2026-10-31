# 🚂 ESPHome DUPLO Train Controller (Experimental)

Control a **LEGO® DUPLO® 10427 may also work with 10428** Bluetooth train from **Home Assistant** using a plain **ESP32**. You only need ESPHome. There are no external components, no MQTT broker, no Bluetooth proxy and no extra server.

The ESP32 finds the train over Bluetooth Low Energy (BLE), connects and pairs with it, and sends commands using the LEGO Wireless Protocol (LWP3). Home Assistant gets buttons for driving, lights and the horn, plus diagnostic sensors for the connection, pairing and battery.

I use a cheap Zigbee wall switch via Zigbee2MQTT and a Home Assistant automation, so my child can drive the train without needing a smartphone.


---

## ✨ Features

- 🔍 **Automatic discovery.** The ESP32 scans for a powered-on newer-generation DUPLO train and reads its address from the live advertisement. No hard-coded MAC address is needed, so rotating private addresses work.
- 🔁 **Auto-reconnect.** While enabled, it starts new discovery windows in the background after boot and whenever the BLE link drops.
- 🔐 **BLE pairing/bonding.** Pairing runs automatically after connecting, or you can trigger it by hand. Existing bonds are kept.
- 🏎️ **Motor control.** Forward (50 %), Reverse (−50 %), Boost (75 %) and Stop.
- 💡 **Light control.** An 11-colour palette as a Home Assistant select, plus Next colour and Light off buttons.
- 📯 **Horn test.** Plays the train's horn sound.
- 🔋 **Battery level.** Read through LWP hub properties and shown as a battery sensor.
- 🩺 **Diagnostics.** Separate binary sensors for *connected*, *authenticated* and *battery reply received*, so you can tell apart "the BLE link is up" and "the train actually answers".
- 🛑 **Fail-safe stop on Wi-Fi loss.** If the ESP32 loses Wi-Fi, it sends a Stop command to the train.
- 🧾 **Verbose, honest logging.** Logs say clearly when something was only *queued locally* and not confirmed by the train.

---

## 🧰 Hardware requirements

| Item | Notes |
|---|---|
| **ESP32 DevKit / ESP32-WROOM-32** | It needs Bluetooth |
| **LEGO DUPLO 10427/10428 train** | A newer-generation DUPLO train that advertises the LEGO hub service. Older DUPLO trains with a different advertisement are ignored. |
| USB cable / power supply | For flashing and running the ESP32. |
| Home Assistant *(optional)* | With the ESPHome integration, for the dashboard controls. |

---

## 🚀 Installation

### 1. Clone the repository

```bash
git clone https://github.com/<your-user>/<your-repo>.git
cd <your-repo>
```

### 2. Create `secrets.yaml`

```yaml
wifi_ssid: "YourNetwork"
wifi_password: "YourWifiPassword"
duplotrain__encryption_key: "BASE64_32_BYTE_KEY"   # generate e.g. with: openssl rand -base64 32
duplotrain__ota_password: "SomeStrongOtaPassword"
```

### 3. Flash the ESP32

```bash
esphome run duplo-train-test.yaml
```

The first flash must be done over USB. Later updates can go over the air (OTA).

### 4. Add it to Home Assistant

Home Assistant should detect the device as **DUPLO Train Test** automatically. If it doesn't, add it with *Settings → Devices & Services → Add Integration → ESPHome* and enter the encryption key from `secrets.yaml`.

---

## 🏠 Home Assistant entities

### Controls

| Entity | Type | Description |
|---|---|---|
| **Auto connect** | Switch | Keeps discovering in the background while the BLE client is idle. Remembered across reboots (default ON). Turning it off cancels a discovery or connection attempt in progress but **does not** drop a link that is already established. |
| **Auto pairing** | Switch | Requests pairing automatically after connecting. Remembered across reboots (default ON). Existing bonds are kept either way. |
| **Requested light colour** | Select | Sets the train light. Shows the **last requested** colour, *not* the actual LED state. |
| **Discover and connect** | Button | Starts one 20 s discovery and connection attempt by hand. |
| **Request pairing** | Button | Forces pairing and re-runs initialization (ignores the *Auto pairing* setting). |
| **Prepare notifications and battery** | Button | Enables notifications again and requests the battery level. |
| **Request battery** | Button | Sends a battery level request. |
| **Forward** | Button | Motor power **+50 %** (continuous). |
| **Reverse** | Button | Motor power **−50 %** (continuous). |
| **Boost** | Button | Motor power **+75 %** (continuous). |
| **Stop** | Button | Motor power **0** and cancels pending initialization. |
| **Disconnect** | Button | Sends Stop, then disconnects. **Turns Auto connect off** so it doesn't reconnect right away. Turn Auto connect back on to resume. |
| **Enable EVENTS mode** | Button | Diagnostic: sets up the sound/light port in EVENTS mode. Not needed for motor control. |
| **Test horn** | Button | Plays the horn sound. |
| **Test light index 07** | Button | Sets the light to *Green* (palette index `0x07`). |
| **Next light colour** | Button | Steps to the next palette colour, wrapping around. |
| **Light off** | Button | Turns the light off. |

> Motor and light commands are refused unless the train is **connected and authenticated**. Diagnostic buttons such as *Request battery*, *Enable EVENTS mode* and *Test horn* only need a connection.

### Diagnostics

| Entity | Type | Meaning |
|---|---|---|
| **Train BLE connected** | Binary sensor | The BLE link is established. |
| **BLE authenticated** | Binary sensor | Connected **and** paired/bonded. |
| **Battery reply received** | Binary sensor | The train answered a battery request. This is the best available sign that the train is really listening. |
| **Train battery** | Sensor (%) | Battery level reported by the train. `unknown` until a reply arrives. |

---

## 🕹️ Typical workflow

1. **Switch on the train.** Keep any other LEGO trains or hubs switched off.
2. With **Auto connect** ON, the ESP32 finds and connects to the train within a few seconds. Otherwise, press **Discover and connect**.
3. Wait until **BLE authenticated** and **Battery reply received** are both ON.
4. Optionally, test the **Light** and **Horn** first.
5. With the train on a safe track, press **Forward**, **Reverse** or **Boost**.
6. Press **Stop** to halt it.
7. Press **Disconnect** when you're done (this also turns Auto connect off).

---

## 🎨 Light colour palette

This config uses the **newer 10427 EVENTS palette**, not the older LPF2 colour palette.

| Option | Byte |
|---|---|
| Off | `0x00` |
| White | `0x01` |
| Green | `0x07` |
| Yellow | `0x08` |
| Light blue | `0x09` |
| Purple | `0x0A` |
| Light pink | `0x0B` |
| Red | `0x0C` |
| Red-pink | `0x0D` |
| Purple-pink | `0x0E` |
| Dark blue | `0x0F` |

Before every colour request, the sound/light port is set up in EVENTS mode first, so no separate setup step is needed.

---

## 📡 Protocol reference

**BLE identifiers**

| Item | Value |
|---|---|
| Manufacturer company ID | `0x0397` (LEGO System A/S) |
| Device type byte (`data[1]`) | `0x21` (newer DUPLO train) |
| Hub service UUID | `00001623-1212-efde-1623-785feabcd123` |
| Hub characteristic UUID | `00001624-1212-efde-1623-785feabcd123` |
| CCCD | `0x2902` |

**Packets used** (LWP3 framing: `[length] [hub ID 0x00] [message type] …`)

| Purpose | Bytes |
|---|---|
| Motor power `P` (port `0x32`) | `08 00 81 32 11 51 00 P` (P = signed −100…100, two's complement) |
| Stop | `08 00 81 32 11 51 00 00` |
| Request battery | `05 00 01 06 05` |
| Battery reply (parsed) | `06 00 01 06 06 XX` (XX = 0…100 %) |
| Enable EVENTS mode (port `0x34`) | `0A 00 41 34 01 01 00 00 00 01` |
| Set light colour `C` | `0B 00 81 34 11 51 01 04 01 C 00` |
| Horn | `0B 00 81 34 11 51 01 07 01 00 00` |

Examples: Forward `+50` → `0x32`, Reverse `−50` → `0xCE`, Boost `+75` → `0x4B`.