# DUPLO 10427: ESPHome BLE control

This firmware runs directly on an ESP32 and exposes train controls to Home
Assistant through the ESPHome native API.

## Requirements

- LEGO DUPLO **10427**, maybe also a **10428**. Older DUPLO trains use different ports/protocol details.
- A USB-powered **original ESP32 development board**, such as an ESP32 DevKit / 
  ESP32-WROOM-32. The YAML uses `board: esp32dev` with ESP-IDF.


## 2. Automatic discovery and connection

Close the LEGO app and stop other Bluetooth controllers from connecting to the train.
Power off other newer DUPLO trains nearby: discovery selects the **first matching
hub family**, not a uniquely configured individual train.

1. Leave **Auto connect** ON (the default).
2. Power on or wake the train using its physical button.
3. The ESP32 finds and connects to it automatically. **Train BLE connected** should
   turn on. No button press or Home Assistant automation is needed.

While disconnected, the ESP32 repeats 20-second discovery windows, with an idle
check every two seconds. It starts after ESP32 boot and resumes after the train
sleeps, loses Bluetooth range, or is powered off. It stops scanning while
connected. Connection attempts have a separate 20-second timeout, after which
automatic discovery can retry.

The two Home Assistant switches are independent and remember their settings
across ESP32 restarts (both default ON on first installation):

| Switch | When ON | When OFF |
| --- | --- | --- |
| **Auto connect** | Keep discovering/reconnecting to an available train | Stop background discovery and pending attempts; leave an established connection alone |
| **Auto pairing** | Request authentication automatically after connecting | Do not make automatic pairing requests; manual pairing remains available |

**Discover and connect** remains a manual one-shot action, including when Auto
connect is OFF. **Disconnect** turns Auto connect OFF before sending Stop and
disconnecting, preventing an immediate reconnection. Turn Auto connect back ON
to resume looking for the train. This also lets you release it to the LEGO app.

No MAC-address editing or second firmware flash is needed. The scan checks the
LEGO manufacturer ID, newer DUPLO system type `0x21`, and LEGO service UUID, then
uses the fresh address **and address type**. This matters because private/random
addresses and sleep-related address changes have been reported for 10427.

**A BLE connection is not proof that the train accepts commands.**

## 3. Automatic pairing and battery check, without moving

The first hardware test worked after **Request pairing**. With **Auto pairing**
ON (the default), the firmware performs that step automatically after connecting:

1. The ESP32 requests pairing, unless the connection is already authenticated,
   and waits up to 15 seconds for authentication to complete.
2. After authentication, it sends Stop, enables notifications again, and requests
   the battery while stationary. It does **not** start motion automatically.
3. Look for **BLE authenticated**, **Battery reply received**, a battery
   percentage, and `Battery reply` in the logs. The expected battery response is
   `06 00 01 06 06 PP` (`PP` = percentage).
4. If initialization fails, **Request pairing** retries the complete sequence.
   **Prepare notifications and battery** and **Request battery** remain available
   as manual diagnostics.

With **Auto pairing** OFF, a new unauthenticated connection is left waiting for
you to press **Request pairing**. That manual button works even while the switch
is OFF. If the connection is already authenticated, notification/battery setup
can proceed without submitting another pairing request. Turning the switch ON
applies to the next connection; use Request pairing to authenticate the current
connection immediately.

The switch controls explicit automatic pairing **requests**, not all Bluetooth
security behaviour. It does not delete existing bonds, disconnect the train,
cancel authentication already submitted, or prevent the train from initiating
authentication. To stop the ESP32 taking the Bluetooth connection, turn **Auto
connect** OFF and press **Disconnect** instead.

A successful pairing request is not pairing completion. **BLE authenticated**
describes this connection, not a persistent bond inventory. The firmware does
not guess a PIN or erase bonds automatically. Each authentication attempt waits
at most 15 seconds; connection discovery continues while Auto connect is ON.
Automatic pairing/recovery still needs testing after sleep and power cycles.

## 4. Continuous movement and Boost

After connecting and pairing, use the buttons directly—no arming or battery
reply is required to enable movement.

| Button | Motor power | Behaviour |
| --- | --- | --- |
| **Forward** | +50 | Continuous forward movement |
| **Reverse** | -50 | Continuous reverse movement |
| **Boost** | +75 | Faster forward movement |
| **Stop** | 0 | Stop the motor |

These match the original repository's default motor-power values; they are not
calibrated speeds. **Boost always means forward +75**, even after Reverse. It
has no expiry: press Forward to return to +50, or Stop to stop. Reverse changes
direction immediately, without the original repository's horn/delay sequence.

There is **no firmware-imposed movement timeout**, no arm switch, and no automatic
Stop on Home Assistant or Wi-Fi loss. Each command replaces the current motor
power. The train's own action tiles, manual controls, battery and BLE behaviour
can still affect movement. Stop requires a working Bluetooth connection.

Reboot/reconnect does not restore a previous movement command or start the train
automatically. Connection initialization, manual pairing/preparation and intentional
disconnect still send Stop; they are not movement timers.
An accepted movement command cancels any still-pending initialization/preparation
so it cannot send a delayed Stop after you start driving.

### Updating from the pulse-test version

Install the new `duplo-train-test.yaml` via OTA, keeping your current secrets.
Use **Forward**, **Reverse**, **Boost** and **Stop** in Home Assistant. The old
**Arm motion tests**, **Forward pulse** and **Reverse pulse** entities were removed;
if Home Assistant retains them as unavailable, remove those obsolete entities
and update any dashboards/automations using them.

## 5. Choose or cycle the light colours

The original **Test light index 07** button sent **only green**, confirmed by the
first hardware test. It was never a colour-cycle command.

The updated firmware adds:

- **Requested light colour**: select any of the ten documented colours, or Off.
- **Next light colour**: advance one step per press, wrapping around through all
  options. Allow about half a second between presses so each colour is visible.
- **Light off**: turn the LED off.
- **Test light index 07**: retained for compatibility; still requests Green.

Each colour request enables the sound/light EVENTS mode, waits 150 ms, then
sends the selected palette value. You no longer need to press **Enable EVENTS
mode** before using the light controls. Pairing must have completed first.
Colour requests do not move the train, and are not restored on reconnect.

The order and 10427 palette values are:

| Option | Hex index |
| --- | --- |
| Off | `00` |
| White | `01` |
| Green | `07` |
| Yellow | `08` |
| Light blue | `09` |
| Purple | `0A` |
| Light pink | `0B` |
| Red | `0C` |
| Red-pink | `0D` |
| Purple-pink | `0E` |
| Dark blue | `0F` |

These are the newer train's documented EVENTS values, not the older generic
LEGO colour table. Indices `02`–`06` are undocumented for this command and are
not sent. Please report any mismatch between a label and the observed colour.

The selector shows the **last requested** colour, not physical LED-state
feedback. Its initial Off label does not prove the train's LED is off.

For the horn, press **Enable EVENTS mode**, then **Test horn**. EVENTS remains a
manual diagnostic for motor troubleshooting, not an assumed universal motor
handshake. The configuration does not subscribe to unknown motor, TAG, or debug
modes.

## 6. Disconnect and recovery

- **Disconnect** disables Auto connect, then attempts Stop before releasing
  Bluetooth. Re-enable Auto connect when you want automatic discovery again.
- With Auto connect ON, simply wake the train again after sleep. Pairing (if
  enabled) and notification setup run again, without replaying any previous
  movement command.
- Stop the train before firmware updates or unplugging the ESP32.
- Bluetooth or ESP32 power loss can prevent Stop from reaching the train; use
  the train's physical controls if the wireless connection is unavailable.

## Troubleshooting / what to report back

- **No matching hub:** confirm Auto connect is ON and the train is a 10427, close the
  LEGO app, check range, and keep other controllers away. Discovery requires both
  its manufacturer payload and service UUID; a different advertisement layout
  may need a code adjustment.
- **Connection fails:** Auto connect retries while enabled; wake the train and
  capture its address and address type from the logs. Stale bonds after sleep are a reported
  possibility; this firmware does not erase bonds automatically.
- **Connected but no battery reply:** copy the pairing, notification/CCCD, write,
  and raw `train_rx` logs. Re-prepare after authentication completes. Some hub
  firmware may need different security settings or a dedicated BLE component.
- **Connected but not authenticated:** check Auto pairing, or press Request
  pairing to authenticate manually without changing the switch setting.
- **Battery works but motion/sound does not:** writes being accepted by the BLE
  stack are not acknowledgments of physical execution. Try the explicit EVENTS
  diagnostic and report the difference; do not add continuous motor retries.
- **Works until sleeping:** report the before/after address, authentication and
  notification logs. Rediscovery should use a fresh address each time.

Report your board model, ESPHome version, train behaviour, battery result,
pairing result, whether EVENTS changed anything, and a short log excerpt.
Remove Wi-Fi credentials, encryption keys and other private information before
sharing logs. Do not share `secrets.yaml`.

## References

- [Original BLE-to-MQTT repository](https://github.com/micschr0/duplo-train-10427-ble2mqtt)
- [ESP32 Arduino reference](https://github.com/romanlamsal/duplo-train-10428-arduino-remote)
- [10427 protocol investigation](https://codecat.io/projects/lego-train/protocol/)
- [Captured 10427 colour commands](https://github.com/corneliusmunz/legoino/issues/90#issuecomment-3456111339)
- [ESPHome BLE client](https://esphome.io/components/ble_client/)
- [ESPHome BLE security](https://esphome.io/components/esp32_ble/)

## Validation status

The updated YAML compiled with **ESPHome 2026.10.0**, ESP-IDF, and
`board: esp32dev`, using synthetic build-only secrets. The user reports that
pairing, light controls and the original movement pulses work on their train.
Automatic rediscovery and the new pairing switch still require hardware testing.
No server was started or device flashed here.
