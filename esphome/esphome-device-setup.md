# ESP32-C3 Development with USB/IP and WSL

This runbook describes how to connect an ESP32 development board connected to a Windows host to an ESPHome container running under Docker in WSL2.

The procedure uses `usbipd-win` to make the USB device available to WSL.

## Prerequisites

* Windows with WSL2
* A Linux distribution installed in WSL
* Docker Desktop with WSL integration enabled
* `usbipd-win` installed on Windows
* An ESP32 board connected by USB
* ESPHome running in Docker

## 1. Identify the USB device

Open PowerShell and run:

```powershell
usbipd list
```

Locate the ESP32 USB device. It should show a USB bus ID and the Espressif USB VID/PID.

Example:

```text
Connected:
BUSID  VID:PID    DEVICE                                  STATE
1-8    303a:1001  USB Serial Device, USB JTAG/serial...  Not shared
```

The bus ID is specific to the current Windows USB connection. **Do not assume it will always be the same.**

## 2. Share the USB device

The first time a particular USB device is used with USB/IP, run PowerShell as Administrator:

```powershell
usbipd bind --busid <BUSID>
```

For example:

```powershell
usbipd bind --busid 1-8
```

Binding normally only needs to be done once for the device.

Verify with:

```powershell
usbipd list
```

The device should now be shown as shared.

## 3. Attach the device to WSL

Run this from a normal PowerShell window:

```powershell
usbipd attach --wsl --auto-attach --busid <BUSID>
```

`--auto-attach` is useful for ESP32 development because the USB device can temporarily disappear and re-enumerate when the ESP32 resets.

Leave this PowerShell process running while using the device.

## 4. Verify the device in WSL

Open the WSL terminal and check that the device is present:

```bash
lsusb
```

For an ESP32-C3, the Espressif USB device should be visible.

Then check the serial device:

```bash
ls -l /dev/ttyACM*
```

A typical result is:

```text
crw-rw---- 1 root dialout ... /dev/ttyACM0
```

The device may not always be assigned exactly the same `/dev/ttyACM*` number after reconnecting.

## 5. Make the device available to Docker

The ESPHome container needs access to the serial device.

For Docker Compose:

```yaml
services:
  esphome:
    # ...
    devices:
      - /dev/ttyACM0:/dev/ttyACM0
```

Use the actual device name reported by WSL.

### Important

Attach the USB device to WSL **before creating the container**.

If the container was created before `/dev/ttyACM0` existed, recreate it after attaching the device:

```bash
docker compose down
docker compose up -d
```

## 6. Use a separate ESPHome test container

A separate ESPHome container is useful when testing a newer ESPHome version without changing the production installation.

Example:

```yaml
services:
  esphome-test:
    image: ghcr.io/esphome/esphome:<ESP_HOME_VERSION>
    container_name: esphome-test
    privileged: true
    restart: "no"
    ports:
      - "6053:6052"
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
    devices:
      - /dev/ttyACM0:/dev/ttyACM0
```

The port mapping is:

```text
Windows/WSL host : container
6053              : 6052
```

This allows the test instance to coexist with an ESPHome instance using port 6052.

The test container can then be accessed through port 6053.

## 7. Flash the ESP32

Once the container is running and the device is mapped:

```bash
docker exec -it esphome-test esphome run /config/<CONFIG_FILE>.yaml
```

ESPHome should detect the serial device:

```text
Connected to ESP32-C3 on /dev/ttyACM0
```

After compilation and flashing, ESPHome should report that the upload completed successfully.

## 8. Monitor the device

The USB serial connection can also be used to monitor the device:

```bash
docker exec -it esphome-test esphome logs /config/<CONFIG_FILE>.yaml
```

This is particularly useful during initial hardware and firmware testing because it does not depend on Wi-Fi being operational.

## 9. USB reset/re-enumeration

An ESP32-C3 using its USB Serial/JTAG interface can reset and temporarily disappear from USB when firmware is flashed or restarted.

With `--auto-attach` enabled, USB/IP should automatically reattach the device after it re-enumerates.

If the device does not reappear:

```powershell
usbipd list
```

Then verify from WSL:

```bash
lsusb
ls -l /dev/ttyACM*
```

If necessary, stop and restart the USB/IP attachment.

## 10. Disconnecting

When finished, stop the ESPHome test container if it is no longer needed:

```bash
docker compose down
```

Then stop the USB/IP attachment from the PowerShell window.

The USB device can subsequently be used normally by Windows.

---

# ESP32-C3 Wi-Fi troubleshooting

During testing of an ESP32-C3 SuperMini, Wi-Fi authentication failed repeatedly with:

```text
Disconnected ... reason='Auth Expired'
```

The failure occurred during Wi-Fi authentication despite a strong signal.

The following tests were performed:

* ESPHome production version: same failure
* Newer ESPHome version: same failure
* WPA2 configuration: confirmed
* Wi-Fi power-save disabled: no improvement
* Board moved away from the Windows laptop: no improvement
* Default/high TX power: authentication failed
* 15 dBm TX power: authentication failed
* 10 dBm TX power: connected immediately

The resulting working configuration was:

```yaml
wifi:
  # ...
  output_power: 10dB
```

This should be treated as a **device/environment-specific workaround**, not as a general ESP32-C3 requirement.

The important observation is that the access point remained clearly receivable at the lower transmit power, so reducing TX power did not make the connection marginal.

If investigating a similar problem, test `output_power` before making disruptive changes to the Wi-Fi infrastructure.

---

# Quick checklist

```text
Windows
  |
  +-- usbipd list
  |
  +-- usbipd bind --busid <BUSID>       # first time only
  |
  +-- usbipd attach --wsl --auto-attach --busid <BUSID>
          |
          v
WSL
  |
  +-- lsusb
  +-- ls -l /dev/ttyACM*
          |
          v
Docker
  |
  +-- devices:
  |     - /dev/ttyACM0:/dev/ttyACM0
  |
  +-- ESPHome
          |
          +-- compile
          +-- flash
          +-- monitor logs
```
