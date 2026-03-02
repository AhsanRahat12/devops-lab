# Keychron Launcher Not Working on Linux (WebHID Fix)

## Problem
When trying to use the Keychron Launcher web app to configure the M3 mouse, the mouse appeared to connect but the settings never loaded — the screen kept prompting to connect. 

This is a permissions issue. By default, Linux does not give regular users access to raw HID USB devices. The browser's WebHID API needs this low-level access to fully communicate with the mouse.

## Fix

1. Create a udev rule for the mouse:
   ```bash
   sudo nano /etc/udev/rules.d/99-keychron.rules
   ```

2. Add the following line:
   ```
   SUBSYSTEM=="hidraw", ATTRS{idVendor}=="3434", MODE="0666"
   ```
   
   **What this does:**
   - `SUBSYSTEM=="hidraw"` — targets raw HID USB devices
   - `ATTRS{idVendor}=="3434"` — filters to only Keychron devices (3434 is Keychron's USB vendor ID)
   - `MODE="0666"` — sets the file permissions so that all users (not just root) can read and write to the device

3. Reload the udev rules:
   ```bash
   sudo udevadm control --reload-rules && sudo udevadm trigger
   ```

4. Unplug and replug the mouse, then try the Launcher again.

## Notes
- The Launcher requires a Chromium-based browser (Chrome, Chromium, Edge, Opera)
- The mouse must be connected via USB cable or 2.4 GHz receiver — Bluetooth will not work with the Launcher
