# Pill Monitor

An Android app that picks up a **KegLand RAPT Pill** hydrometer's readings over Bluetooth, with no
RAPT cloud account needed. It logs the data on your phone and shows it on a dashboard.

- **Live dashboard:** gravity (SG or °Plato), ABV, apparent attenuation, temperature, fermentation
  velocity, battery and signal strength.
- **Interactive trend charts:** gravity and temperature over 24 h, 3 days, 7 days or the whole
  brew. Press and drag on a chart to read exact values.
- **Background monitoring:** a foreground service keeps listening with the screen off and shows the
  latest reading in its notification.
- **Smart listening sensitivity:** full sensitivity while the Pill is in range. After an hour without
  a broadcast (or two missed reports on hourly reporting) it drops to low power, and it switches back
  as soon as the Pill is heard again. Tap the chip under the status line to boost or change modes.
- **"Next reading" countdown:** the Pill can't be asked for a reading on demand, so the app tells you
  when the next broadcast is due.
- **Temperature alerts:** set a range in Settings and get a notification when the fermenter leaves
  it, with reminders every 4 hours and an all-clear when it recovers.
- **Brew sessions:** start a new brew per Pill, enter a measured OG (or let the app use the first
  reading), and rename Pills. Several Pills are supported.
- **Private:** everything stays in a database on the phone.

<p>
  <img src="docs/dashboard-dark.png" width="240" alt="Dashboard, dark theme">
  <img src="docs/dashboard-light.png" width="240" alt="Dashboard, light theme">
  <img src="docs/searching-dark.png" width="240" alt="Searching for a Pill">
</p>

*Screens rendered from a simulated three-day ale fermentation.*

## How it works

The Pill doesn't accept Bluetooth connections. In Bluetooth mode it wakes on its interval,
*broadcasts* each measurement as a BLE advertisement, and goes back to sleep with its radio off. The
app only listens, so it never affects the Pill's battery, and it can't make the Pill report early. Each broadcast carries manufacturer data
that starts with `RAPT`. Android reads the first two bytes (`RA` = `0x4152`) as the company ID and
hands over the remaining 23 bytes:

| Bytes | v1 payload | v2 payload |
|---|---|---|
| 0–1 | `"PT"` | `"PT"` |
| 2 | version = 1 | version = 2 |
| 3–8 | Pill MAC address | reserved (1), velocity valid (1), gravity velocity f32 (4) |
| 9–10 | temperature u16 → `raw / 128 − 273.15` °C | same |
| 11–14 | gravity f32 → `raw / 1000` SG | same |
| 15–20 | accelerometer x, y, z i16 → `raw / 16` | same |
| 21–22 | battery i16 → `raw / 256` % | same |

All values are big-endian. A second broadcast under company ID `0x454B` (`KE`) carries `G` followed
by the firmware version string. The decoder is in
[`RaptPillParser.kt`](app/src/main/java/io/github/pillmonitor/ble/RaptPillParser.kt). Its unit tests
use real Pill payloads from the [rapt-ble](https://github.com/sairon/rapt-ble) project, which Home
Assistant uses to read the same broadcasts.

## Setting up the Pill

1. Open the Pill's own setup page: lift it on and off its charging pad until the green light
   flashes, then join the Wi-Fi hotspot it creates. Under **Settings → Telemetry method**, choose
   **Bluetooth**. (The RAPT cloud portal can't change these settings, because a Pill in Bluetooth
   mode never goes online.)
2. Pick a reporting interval of 15, 30 or 60 minutes (60 is the default and gives the longest
   battery life). The app learns the interval from the readings it receives.
3. Keep the phone within about 10 m. Stainless steel fermenters weaken the signal a lot.

## Installing

The quickest option is to sideload the prebuilt APK, `PillMonitor-1.0.apk`, from this folder. Copy
it to your phone (Android 8.0+ with Bluetooth LE), open it, and allow installs from that source when
Android asks. Then grant the *Nearby devices* permission, plus *Location* on Android 11 and older.

## Building

**Android Studio:** choose **File → Open**, select this folder, wait for Gradle sync to finish, then
press **Run** with your phone connected over USB debugging.

**Command line:** with `JAVA_HOME` set to a JDK 17 and `sdk.dir` in `local.properties` pointing at
an Android SDK:

```bash
./gradlew testDebugUnitTest assembleRelease
```

The APK is written to `app/build/outputs/apk/release/app-release.apk`. Release builds are signed
with the debug key so they can be sideloaded for personal use.

Unit tests for the decoder and brew math are under `app/src/test`.

### Tips

- If the app misses readings, turn on **High-sensitivity scanning** in Settings.
- Some phone makers (Xiaomi, Huawei, Samsung and others) aggressively kill background apps. For
  monitoring that runs for days, set the app's battery usage to **Unrestricted** in Android
  settings.

## Project layout

```
app/src/main/java/io/github/pillmonitor/
├── ble/        RaptPillParser (decoding), RaptScanner (BLE scan → Flow), BlePermissions
├── data/       Room database, PillRepository (de-duplication + storage), settings
├── service/    PillScanService: foreground service that runs the scan
├── ui/         Compose dashboard, settings sheet, dialogs, theme, chart components
└── util/       BrewMath (ABV, attenuation, Plato, velocity), formatting
```
