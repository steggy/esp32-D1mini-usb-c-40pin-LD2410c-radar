# ESP32 D1 Mini LD2410C Radar

An ESPHome mmWave presence sensor built with an ESP32 D1 Mini-style USB-C
40-pin board and a Hi-Link HLK-LD2410C radar module.

The device detects moving and stationary targets and reports their distances to
Home Assistant. Home Assistant automations are intentionally not included;
each user can decide what the sensor should control.

## Features

- Overall presence detection
- Moving-target detection
- Stationary-target detection
- Moving-target distance
- Stationary-target distance
- Overall detection distance
- Selectable LD2410C distance resolution
- Radar restart and parameter-refresh controls
- Wi-Fi provisioning through:
  - Captive portal
  - Bluetooth
  - USB serial
- Automatic ESPHome Device Builder adoption
- Five-power-cycle factory reset
- Tested snap-fit 3D-printed enclosure

## Tested hardware

The firmware, wiring, and enclosure were tested with these exact components:

| Component | Tested hardware | Purchase link |
|---|---|---|
| ESP32 | ACEIRMC Type-C D1 Mini NodeMCU ESP32 ESP-WROOM-32, USB-C, 40-pin | [Amazon](https://www.amazon.com/dp/B0D1V336DL) |
| Radar | XIITIA/Hi-Link HLK-LD2410C 24 GHz mmWave radar with UART, Bluetooth, and pin header | [Amazon](https://www.amazon.com/dp/B0CH8CRDS6) |
| Documentation | Official Hi-Link HLK-LD2410C product information and downloads | [Hi-Link](https://www.hlktech.net/index.php?id=1095) |

The enclosure was designed and tested with these exact physical components.
Similar ESP32 boards or LD2410C modules may run the firmware, but enclosure fit,
USB connector position, and pin placement are not guaranteed.

Additional parts:

- Four connecting wires
- USB-C data cable for programming
- USB-C power supply and cable
- 3D-printed enclosure

## Wiring

| LD2410C | ESP32 | Purpose |
|---|---|---|
| VCC | 5V/VCC | Radar power |
| GND | GND | Ground |
| TX | GPIO16/RX | Data from radar |
| RX | GPIO17/TX | Data to radar |

TX and RX must be crossed:

```text
LD2410C TX -> ESP32 GPIO16
LD2410C RX -> ESP32 GPIO17
```

The UART configuration is:

```text
Baud rate: 256000
Data bits: 8
Parity:    None
Stop bits: 1
```

## ESPHome entities

The configuration exposes:

- Presence
- Moving Target
- Still Target
- Detection Distance
- Moving Distance
- Still Distance
- Distance Resolution
- Restart Radar Module
- Refresh Radar Parameters

Home Assistant may also provide an ESPHome firmware update entity.

The Presence entity has a two-second delayed-off filter to reduce rapid
on/off transitions.

## Configuration files

Two ESPHome configurations are provided:

### `radar.yaml`

Use this version when compiling from the Linux terminal. It requires a private
`secrets.yaml` containing:

- API encryption key
- OTA password
- Setup access-point password

The completed `secrets.yaml` must never be committed to Git.

### `radar-factory.yaml`

This is the secret-free configuration used to build public factory firmware or
preload an ESP32.

It contains:

- No home Wi-Fi credentials
- No API encryption key
- No OTA password
- No private setup password

After Wi-Fi provisioning, ESPHome Device Builder can discover and adopt the
device using the public configuration in this repository.

## Method 1: Preloaded device or factory firmware

This is the recommended Home Assistant ESPHome Device Builder method.

### Provision Wi-Fi

1. Power the radar using USB-C.
2. Wait up to 90 seconds for the setup network to appear.
3. Connect a phone, tablet, or computer to:

   ```text
   Radar Setup
   ```

4. The public factory setup network is intentionally open.
5. If the captive portal does not open automatically, browse to:

   ```text
   http://192.168.4.1/
   ```

6. Select the owner’s 2.4 GHz Wi-Fi network.
7. Enter the Wi-Fi password and save it.
8. The radar will disconnect from the setup network and join the selected
   network using DHCP.

Wi-Fi can also be provisioned through Bluetooth or USB using an
ESPHome Improv-compatible installer.

### Adopt in ESPHome Device Builder

After the radar joins the network:

1. Open ESPHome Device Builder.
2. Wait for `ld2410c-radar-XXXXXX` to appear.
3. Select **Adopt**.
4. Choose the desired device name.
5. Allow ESPHome to compile and install the adopted configuration.
6. Add the discovered ESPHome device to Home Assistant.

The MAC-address suffix prevents hostname collisions when multiple units use the
same factory firmware.

The public source used during adoption is:

```text
github://steggy/esp32-D1mini-usb-c-40pin-LD2410c-radar/radar-factory.yaml@master
```

## Method 2: Compile and install from Linux

### Requirements

The `esphome`, `esptool`, and `openssl` commands must already be installed.

Verify them with:

```bash
esphome version
esptool version
openssl version
```

### Clone the repository

```bash
git clone https://github.com/steggy/esp32-D1mini-usb-c-40pin-LD2410c-radar.git
cd esp32-D1mini-usb-c-40pin-LD2410c-radar
```

### Create private secrets

Generate unique credentials for the physical device:

```bash
umask 077

radar_api_key="$(openssl rand -base64 32)"
radar_ota_password="$(openssl rand -hex 16)"
radar_setup_password="$(openssl rand -hex 8)"

printf 'api_encryption_key: "%s"\nota_password: "%s"\nsetup_ap_password: "%s"\n' \
  "$radar_api_key" \
  "$radar_ota_password" \
  "$radar_setup_password" \
  > secrets.yaml

chmod 600 secrets.yaml
unset radar_api_key radar_ota_password radar_setup_password
```

Alternatively, copy `secrets.example.yaml` to `secrets.yaml` and replace every
placeholder manually.

Never commit or share the completed `secrets.yaml`.

### Validate the configuration

```bash
esphome config radar.yaml
```

Continue only when ESPHome reports:

```text
INFO Configuration is valid!
```

### Compile the firmware

```bash
esphome compile radar.yaml
```

The first compilation may take several minutes while ESPHome downloads and
builds the ESP32 toolchain.

### Identify the ESP32 serial port

Connect the ESP32 using a data-capable USB-C cable:

```bash
ls -l /dev/serial/by-id/
```

Use the complete stable device path:

```bash
radar_port=/dev/serial/by-id/usb-YOUR-ESP32-SERIAL-DEVICE
```

Verify the selected board:

```bash
esptool --port "$radar_port" flash-id
```

Do not use a guessed `/dev/ttyUSB0` when a stable `/dev/serial/by-id/` path is
available.

If serial access is denied, add the Linux account to the `dialout` group and
then log out and back in:

```bash
sudo usermod -aG dialout "$USER"
```

### Install over USB

```bash
esphome upload radar.yaml --device "$radar_port"
```

Watch the startup log:

```bash
esphome logs radar.yaml --device "$radar_port"
```

Exit the log with `Ctrl-C`.

### Provision Wi-Fi

1. Connect to the `Radar Setup` Wi-Fi network.
2. Enter the `setup_ap_password` from the private `secrets.yaml`.
3. Browse to `http://192.168.4.1/` if the portal does not open.
4. Select the desired 2.4 GHz Wi-Fi network.
5. Enter its password and save it.

### Add to Home Assistant

Home Assistant should discover the ESPHome device automatically.

If it does not:

1. Open **Settings → Devices & services**.
2. Select **Add integration**.
3. Choose **ESPHome**.
4. Enter the radar’s IP address or `.local` hostname.
5. Enter the `api_encryption_key` from the private `secrets.yaml`.

## Later updates

Validate and compile an update:

```bash
esphome config radar.yaml
esphome compile radar.yaml
```

Install it over USB:

```bash
esphome upload radar.yaml --device "$radar_port"
```

Or install it over the network:

```bash
esphome run radar.yaml --device DEVICE-IP-OR-HOSTNAME
```

Do not erase the flash for an ordinary update.

## Factory reset

Five quick power cycles clear stored Wi-Fi settings and ESPHome preferences:

1. Turn the unit on and off five times.
2. Keep each powered-on interval below ten seconds.
3. Power the unit on normally.
4. Wait for `Radar Setup` to reappear.
5. Provision Wi-Fi again.

Factory reset does not replace the installed firmware.

## Enclosure

The tested Bambu Studio 3MF project is:

```text
3d-box/esp32-d1mini-usb-c-40pin-ld2410c-radar.3mf
```

The enclosure is designed for:

- ESP32 D1 Mini-style USB-C 40-pin board
- HLK-LD2410C radar
- Snap-fit lid
- Spring retention system
- Access to the USB-C connector

The radar antenna must face the intended detection area.

## Security

- Never commit `secrets.yaml`.
- Never publish firmware compiled from `radar.yaml`.
- Never publish API keys, OTA passwords, setup passwords, or Wi-Fi credentials.
- Only firmware compiled from `radar-factory.yaml` is intended for public use.
- Provision an open factory device promptly.
- Adopt the device into the owner’s ESPHome Device Builder for ongoing control.
- Use unique credentials for every privately compiled physical device.

## Project files

```text
.
├── .gitignore
├── 3d-box/
│   └── esp32-d1mini-usb-c-40pin-ld2410c-radar.3mf
├── README.md
├── radar-factory.yaml
├── radar.yaml
└── secrets.example.yaml
```

Generated build directories and the private `secrets.yaml` are intentionally
excluded from Git.
