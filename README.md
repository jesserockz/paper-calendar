# Paper Calendar

A battery powered Home Assistant calendar for e-paper displays, built with [ESPHome](https://esphome.io).

The device fetches your Home Assistant calendar events with the `calendar.get_events` action, draws them in a day or 3-day view, and then deep-sleeps until the next refresh or button press, so a single charge lasts a long time.

![Day view](static/screenshots/day-view.png)

## Supported devices

| Device                                                                                    | Chip     | Config                               |
|-------------------------------------------------------------------------------------------|----------|--------------------------------------|
| [Seeed Studio reTerminal E1001](https://www.seeedstudio.com/reTerminal-E1001-p-6534.html) | ESP32-S3 | [reterminal-e1001](reterminal-e1001) |

Each supported device has its own folder with its own prebuilt firmware, installable from the [web installer](https://jesserockz.github.io/paper-calendar/). All devices run the same firmware name (`paper-calendar`) and share the same rendering and features.

## Features

- Day and 3-day views of any number of Home Assistant calendars, merged and sorted.
- Deep sleep between refreshes with a configurable interval, awake time, and an overnight downtime window.
- Buttons wake the device *and* perform their action: page through the days or switch the view. A short beep confirms the press registered, since e-paper takes a few seconds to redraw.
- All-day events, word-wrap or truncation for long titles, an ignore list for events you do not want shown, 12/24-hour clock, and a battery readout (percentage, voltage, or both) in the header.
- Everything is configured through entities on the device itself - no Home Assistant helpers, template sensors, or YAML edits needed.

![3-day view](static/screenshots/three-day-view.png)

## Installation

Open the [web installer](https://jesserockz.github.io/paper-calendar/) in Chrome or Edge, plug the device in via USB, and click the install button for your device. The same flow lets you provision your Wi-Fi credentials over USB (via [Improv Serial](https://www.improv-wifi.com/)).

If you skip that, there are two other ways to get the device onto Wi-Fi, both only active while no Wi-Fi credentials are saved:

- **Bluetooth** ([Improv over BLE](https://www.improv-wifi.com/)): set it up from your phone or from Home Assistant's discovered devices.
- **Fallback hotspot**: the device opens an open hotspot named `paper-calendar-xxxxxx` (a MAC address suffix). Its screen shows a QR code to join it; then open `192.168.4.1` to pick your network.

No credentials are baked into the firmware: Home Assistant provisions the API encryption key automatically when the device is adopted, and every unit gets a unique hostname (a MAC address suffix).

## First-boot setup

The device stays awake until setup is confirmed, so there is no race against deep sleep. A device without saved Wi-Fi credentials first shows the Wi-Fi setup screen (see [Installation](#installation)):

![Wi-Fi setup screen](static/screenshots/wifi-setup.png)

Once it is connected, it moves on to the Home Assistant setup screen:

![Setup screen](static/screenshots/setup-screen.png)

1. Home Assistant discovers the device - add it via the **ESPHome** integration.
2. On the device page, press **Configure** and tick **Allow the device to perform Home Assistant actions** (the device calls `calendar.get_events`, which needs this).
3. Fill in the **Calendars** text entity with a comma separated list of calendar entity ids, e.g. `calendar.family, calendar.work`.
4. Optionally adjust the other options below.
5. Press the **Confirm setup** button. The display refreshes and the device starts its normal sleep cycle.

## Configuration entities

All of these live on the device and persist across deep sleep and reboots:

| Entity               | Type   | What it does                                                                                                                                                                            |
|----------------------|--------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Calendars            | Text   | Comma separated calendar entity ids to show, e.g. `calendar.family, calendar.work`.                                                                                                     |
| Ignored events       | Text   | Comma separated event titles to hide, matched case-insensitively against the full title, e.g. `Out of office, Lunch`.                                                                   |
| Long event names     | Select | `Truncate` cuts long titles with `~`; `Word wrap` wraps them onto up to three lines at word boundaries.                                                                                 |
| Battery display      | Select | Header battery readout: `Percentage`, `Voltage`, or `Both`.                                                                                                                             |
| Refresh interval     | Number | Minutes asleep between display refreshes (5 to 720).                                                                                                                                    |
| Awake time           | Number | Seconds the device stays awake after boot or the last button press (10 to 300).                                                                                                         |
| 12-hour time         | Switch | Show times as `6:30pm` instead of `18:30`.                                                                                                                                              |
| Downtime start / end | Time   | Refresh wakes that would land inside this window are deferred to its end, e.g. 22:00 to 06:30 stops overnight refreshes. Buttons still wake the device. Equal times disable the window. |
| Prevent sleep        | Switch | Keeps the device awake, e.g. while you install an OTA update.                                                                                                                           |
| Confirm setup        | Button | Finishes first-boot setup; the device refuses to sleep until this has been pressed (and Calendars is filled in).                                                                        |

The device also exposes **Battery** (%), **Battery voltage**, an **Awake** binary sensor (Home Assistant keeps the last state while the device sleeps instead of marking it unavailable), and - on the factory firmware - a **Firmware** update entity that offers new releases from this repository.

## Buttons

On the reTerminal E1001, the three top buttons are:

| Button | Press   | Action                                                                              |
|--------|---------|-------------------------------------------------------------------------------------|
| Left   | Short   | Page one view back (1 or 3 days, depending on the view).                            |
| Right  | Short   | Page one view forward.                                                              |
| Green  | Short   | Cycle the view: day <-> 3-day.                                                      |
| Green  | Hold 1s | Jump back to today (only while the device is awake; a wake always starts on today). |

Any button also wakes the device from deep sleep, and the wake press performs its normal action (paging starts from today; the green hold-for-today only works while already awake).

## Updates

The factory firmware checks this repository's GitHub releases and exposes a **Firmware** update entity in Home Assistant. Because the device sleeps most of the time, the easiest way to update is: turn on **Prevent sleep** (or press a button to wake it), install the update from the update entity, then turn **Prevent sleep** back off.

If you have adopted the device into your own ESPHome dashboard, you update it from there instead, like any other ESPHome device.

## Using this config as a package

### Adopting into the ESPHome dashboard

The factory firmware advertises the device's core config (e.g. [reterminal-e1001/reterminal-e1001.yaml](reterminal-e1001/reterminal-e1001.yaml)) for adoption. Adopting creates a local config in your dashboard that imports this repository as a package, with your own Wi-Fi secrets and API key on top, and without the installer-only extras from the `.factory.yaml` wrapper.

### Personal overlay with fixed credentials

For a fully pinned build (fixed Wi-Fi, fixed API key, plain hostname), include the core config as a package and add your credentials on top:

```yaml
# my-paper-calendar.yaml
packages:
  paper_calendar: github://jesserockz/paper-calendar/reterminal-e1001/reterminal-e1001.yaml@main

esphome:
  # One personal device: keep the plain hostname
  name_add_mac_suffix: false

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  # Connect straight to the stored AP instead of scanning first - faster
  # wake-to-refresh, which matters on battery.
  fast_connect: true

api:
  encryption:
    key: !secret api_encryption_key
```

## Development

The configuration is plain ESPHome YAML (requires ESPHome 2026.9.0 or newer). Everything device-agnostic lives in [common/](common); each device folder holds only that device's hardware:

- [common/paper-calendar.yaml](common/paper-calendar.yaml) - the calendar itself: configuration entities, fetch / sleep / button logic, Wi-Fi setup, and the API, OTA, Wi-Fi and time setup. Every device includes it as a package, and every device uses the firmware name `paper-calendar`.
- [common/factory.yaml](common/factory.yaml) - the web-installer extras: Improv Serial and Bluetooth provisioning, dashboard import, and GitHub-release OTA updates.
- [common/calendar-render.yaml](common/calendar-render.yaml) and [common/fonts.yaml](common/fonts.yaml) - the render lambda and fonts, shared by every device config and the screenshot tool.
- `<device>/<device>.yaml` - the device config (what gets adopted): `common/paper-calendar.yaml` plus the device's hardware, e.g. [reterminal-e1001/reterminal-e1001.yaml](reterminal-e1001/reterminal-e1001.yaml).
- `<device>/<device>.factory.yaml` - the web-installer image for that device: its device config plus `common/factory.yaml`, and the project name and version.
- [screenshots/screenshots.yaml](screenshots/screenshots.yaml) - a host-platform config that renders the display with demo events and saves the screenshots used on this page (`esphome run screenshots/screenshots.yaml`, then convert the BMPs from `screenshots/.esphome/snapshots/` to PNG in `static/screenshots/`).

```bash
esphome compile reterminal-e1001/reterminal-e1001.factory.yaml
```

Releases are built automatically: pushes to `main` update a draft release via Release Drafter, the Build workflow builds every device group listed in [.github/workflows/build.yml](.github/workflows/build.yml) and attaches one manifest per device, and publishing the release deploys the [installer site](https://jesserockz.github.io/paper-calendar/).

### Adding a device

Create a `<device>/` folder with two files, modelled on [reterminal-e1001](reterminal-e1001):

`<device>/<device>.yaml` includes `../common/paper-calendar.yaml` under `packages:` and adds only hardware:

- the `esp32:` block, and a `logger:` override if the USB serial is not the default port;
- the display with `id: epaper` and `lambda: !include ../common/calendar-render.yaml`;
- the battery voltage sensor with `id: battery_voltage`, in volts at the battery (the common package derives the **Battery** percentage from it);
- the wake sources under `deep_sleep:` (the common package sets its `id: sleeper`), with `on_wake` calling the `wake_page_back`, `wake_page_forward` or `wake_cycle_view` scripts;
- the buttons, calling the `action_page_back`, `action_page_forward`, `action_cycle_view` and `action_today` scripts;
- optionally, a `view_button` substitution naming the view / back-to-today button for the footer hint on the display (defaults to `middle`);
- these hook scripts, which the common package calls but never defines:

| Hook script         | Called                                                        | reTerminal E1001                        |
|---------------------|---------------------------------------------------------------|-----------------------------------------|
| `feedback_click`    | On a button or wake press                                     | Buzzer click                            |
| `feedback_home`     | On the jump back to today                                     | Buzzer chirp                            |
| `feedback_ok`       | When **Confirm setup** is accepted                            | Buzzer chime                            |
| `feedback_error`    | When **Confirm setup** is refused (Calendars empty)           | Buzzer double beep                      |
| `read_battery`      | Once per wake, from `on_boot`; must update `battery_voltage`  | Powers the divider, samples, powers off |
| `before_deep_sleep` | Right before `deep_sleep.enter`, which waits for it to finish | Nothing (`then: []`)                    |

A device without a buzzer defines the `feedback_*` scripts as `then: []`. Keep any `esphome: on_boot:` in list form: package lists are concatenated, but a dict-form `on_boot` would replace the common one.

`<device>/<device>.factory.yaml` includes the device config and `../common/factory.yaml` with the folder name as the `device` var (used for the dashboard import URL and the `firmware/<device>.manifest.json` update manifest), and carries the `esphome: project:` block itself, since the build workflow stamps the release version into the file it builds. Then add a build group for the folder in [.github/workflows/build.yml](.github/workflows/build.yml), an install button in [static/index.md](static/index.md), and a row in [Supported devices](#supported-devices).
