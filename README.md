# Quattro ZBD – Android Control App

Android remote control for the Quattro ZBD quadruped robot. It talks to the robot's ESP32 over Wi-Fi (UDP): virtual joysticks and buttons go out, battery / IMU / mode telemetry comes back. It can also use a phone-as-puppet mode, where the robot mirrors the tilt of your phone.

Firmware repository: _add link_

---

## Features

- Two virtual joysticks: **Move** (forward/back + strafe) and **Spin** (turn on the spot)
- On-screen **L1 / R1 / L2 / R2 / A / B** buttons, mapped like a gamepad
- Live telemetry: robot state, battery % and voltage, accelerometer, balance and gait (TROT / WALK)
- **Emote mode** screen: pick and play emotes (the robot owns the list; the app mirrors the robot's own menu buttons)
- **Puppet mode**: the robot copies your phone's roll, pitch and yaw rate; **ZERO** sets the way you are holding the phone as neutral
- **Recall** button to re-run the robot's IMU auto-calibration (standing only)
- Physical gamepad buttons plugged into / paired with the phone (A, B, L1, R1, L2, R2 and sticks) also work
- Screen stays on while the app is open; puppet mode switches off automatically if the app goes to the background

---

## Requirements

| | |
|---|---|
| Android Studio | Recent stable release (Koala or newer recommended) |
| Language / UI | Kotlin, Jetpack Compose (Material 3) |
| Phone | Android phone with Wi-Fi. A gyroscope / rotation-vector sensor is only needed for puppet mode |
| Robot | Quattro ZBD running the matching ESP32 firmware |

Libraries are the standard AndroidX / Compose set that a new Android Studio **Empty Activity (Compose)** project already includes (`androidx.activity:activity-compose`, Compose Material 3, lifecycle runtime, Kotlin coroutines). There are no third-party dependencies.

The manifest needs the normal network permission:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

---

## Build & install

1. Clone this repository and open the project folder in Android Studio.
2. Wait for **Gradle sync** to finish (sync is separate from build: if the Run button is grey, sync and device selection are usually the cause).
3. Enable **Developer options → USB debugging** on the phone and connect it by USB. Accept the authorization prompt on the phone.
4. Select the phone in the device dropdown and press **Run**.

### If you fork or rename the app

The `namespace` and `applicationId` in `app/build.gradle` and the `package` line at the top of `MainActivity.kt` must all agree, for example:

```kotlin
// build.gradle (app)
namespace = "com.example.quattro_zbd"
applicationId = "com.example.quattro_zbd"
```

```kotlin
// MainActivity.kt, first line
package com.example.quattro_zbd
```

A stale `package` line makes the app install but close immediately on launch. The launcher name comes from `app_name` in `res/values/strings.xml`. Using a different `applicationId` lets this app be installed next to the older hexapod app.

---

## Connecting to the robot

1. Power the robot. The ESP32 creates its own Wi-Fi network:

   | Setting | Value |
   |---|---|
   | SSID | `QUADRUPED_ESP32` |
   | Password | `12345678` (default, change it in the firmware) |
   | Robot IP | `192.168.4.1` |
   | UDP port | `5000` |

2. On the phone go to **Wi-Fi settings** and join `QUADRUPED_ESP32` manually. The app does not do this for you.
3. If Android says the network has no internet and offers to switch to mobile data, **stay on the robot's network**.
4. Open the app. When telemetry appears (battery, state) the link is working.

The IP and port are constants at the top of `MainActivity.kt` (`esp32Ip`, `udpPort`). If you change the Wi-Fi name or password in the firmware, only the phone's Wi-Fi settings need to change; the app itself doesn't hold them.

> If the phone is still trying to join the old hexapod network name (`HEXAPOD_ESP32`), nothing will appear to be wrong in the app but no data will arrive.

---

## Controls

| Control | Action |
|---|---|
| **Move** joystick | Walk forward/back and strafe |
| **Spin** joystick | Turn on the spot |
| **L1 / R1** | Body height down / up (hold both for 2 s = stand / sit) |
| **L2 / R2** | Pitch trim nose down / up |
| **A** | Body leveling on/off |
| **B** | Gait toggle: trot ↔ walk |
| **EMOTE** | Enter / leave emote mode |
| **RECALL** | Re-run IMU auto-calibration (standing only) |
| **PUPPET** | Phone-tilt control on/off (hold the phone upright, top = nose) |
| **ZERO** | In puppet mode, take the current phone attitude as neutral |

**Emote mode screen:** L1 / R1 move the selection on the robot, **A** plays, **B** exits. These are the same four buttons as the robot's own screen menu.

**Menus on the robot's screen:** while the robot is sitting, L1/R1 = up/down, L2/R2 = left/right, A = open/save, B = back/cancel.

---

## Telemetry shown

| Item | Notes |
|---|---|
| State | SLEEP, ACTIVE, BODY LEVELING, PUPPET, CALIBRATION, EMOTE MODE |
| Battery | Percentage estimated linearly from 10.0 V (0 %) to 12.6 V (100 %), plus the raw voltage. Suited to a 3S Li-ion/LiPo pack; adjust in `MainActivity.kt` for other packs |
| Accelerometer | X / Y / Z in m/s² |
| Balance / gait | Balance ON/OFF and TROT/WALK |
| Joint / offset | Used while editing servo offsets from the robot menu |

---

## Protocol (for developers)

All values little-endian, over UDP to `192.168.4.1:5000`, about every 8 ms.

**App → robot (61 bytes)**

| Bytes | Content |
|---|---|
| 0–3, 4–7, 8–11 | joystick forward, side, spin (float) |
| 12–15, 16–19 | left / right trigger values (float) |
| 20 | buttons 1: bit0 A, bit1 B, bit2 L1, bit3 R1, bit4 emote toggle, bit5 emote play, bit6 emote stop |
| 21 | selected emote index |
| 22 | buttons 2: bit0 manual IMU calibration, bit6 L2, bit7 R2 |
| 23–47 | reserved (0) |
| 48 | puppet flags: bit0 puppet on, bit1 re-zero |
| 49–60 | phone roll (deg), pitch (deg), gyro Z (deg/s) as floats |

**Robot → app (28 bytes)**

| Bytes | Content |
|---|---|
| 0–11 | accelerometer X, Y, Z (float) |
| 12–15 | battery voltage (float) |
| 16–19 | selected joint ID (int) |
| 20–23 | joint offset (float) |
| 24 | display state (1 active, 2 calibration, 3 emote, otherwise sleep) |
| 25 | balance on |
| 26 | walk gait on |
| 27 | playing emote index |

The emote list in the app (`emoteNames` in `MainActivity.kt`) must match the firmware `EMOTES[]` order, because the app sends the index:

Curious Head Tilt, Cautious Object Tap, The Wiggle, Play Bow, Happy Dance into Sneak, Breathing into Foot Stomp, Matrix Gyro Roll, Push-Ups, Sit & Wave Hello.

If the firmware's emote list changes, update `emoteNames` to match.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Run button greyed out | Gradle sync finished? Phone selected in the device dropdown? USB debugging authorized? |
| App installs, then bounces back to home screen | `package` line in `MainActivity.kt` doesn't match `namespace` / `applicationId` |
| App opens but no telemetry | Phone joined `QUADRUPED_ESP32`? Mobile data auto-switch off? Robot powered and booted? |
| Robot doesn't respond to sticks | Robot must be standing (hold L1 + R1 for 2 s). Release the emergency stop if used |
| Stuck on the emote screen | Press **B**; the app leaves the screen immediately and re-syncs with the robot's next telemetry packet |
| Puppet mode jumpy | Press **ZERO** while holding the phone in your neutral pose; check the phone has a rotation-vector sensor |

---

## Safety

The robot can fall and servos can pinch. Test on a stand first, keep the robot's emergency stop (SELECT/BACK on a gamepad) in mind, and keep the app in the foreground while driving. If no control packets arrive for about half a second, the firmware's stale-data failsafe takes over.

## License

Add your license here (e.g. MIT).
