# ESPHome Device Setup

## Initial USB Flash

### 1. Connect the device

Connect the ESP32 to Windows via USB.

### 2. Identify the USB device

In an elevated PowerShell:

    usbipd list

Identify the ESP32 by its Espressif VID/PID, typically 303a:1001.

Example:

    1-8    303a:1001    USB Serial Device (COM5), USB JTAG/serial debug unit

The BUSID is device-dependent and may change when another device is connected.

### 3. Bind the device

If the device is not already shared:

    usbipd bind --busid <BUSID>

### 4. Attach it to WSL

    usbipd attach --wsl --busid <BUSID>

Verify in Ubuntu:

    lsusb
    ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null

For the ESP32-C3 SuperMini, this normally appears as:

    /dev/ttyACM0

### 5. Make USB re-enumeration automatic

For ESP32 devices using native USB/JTAG, flashing/resetting can cause the USB device to disappear and re-enumerate.

Start:

    usbipd attach --wsl --auto-attach --busid <BUSID>

Leave this command running while flashing.

This also handles unplug/replug of the device.

### 6. Make the USB device available to Docker

The ESPHome Compose service needs the serial device mapped:

    devices:
      - /dev/ttyACM0:/dev/ttyACM0

Recreate the container after changing Compose:

    docker compose up -d

Verify:

    docker exec -it esphome ls -l /dev/ttyACM0

### 7. Determine the ESP32 target

For an unknown board, don't assume the ESP32 variant.

The ESP32-C3 SuperMini used here requires:

    esp32:
      board: esp32-c3-devkitm-1
      framework:
        type: esp-idf

### 8. Initial flash

From the ESPHome config directory:

    docker exec -it esphome esphome run <device>.yaml

ESPHome should discover the serial device and perform the initial USB flash.

### Notes

- `usbipd attach --auto-attach` is currently started manually. Automating it can be considered later if this becomes frequent.
- The BUSID is not a stable device identifier; determine it with `usbipd list` when switching devices.
- `/dev/ttyACM0` is currently used by the Docker mapping, but the serial device name may need revisiting if multiple USB serial devices are used simultaneously.