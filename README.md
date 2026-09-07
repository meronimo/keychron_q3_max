# Keychron Q3 / Q3 Max - RGB Fix (static, Yellow, full brightness)

Sets the RGB matrix lighting to static Yellow / 100% brightness over the
cable and saves it permanently to the keyboard's EEPROM. After that, the
setting persists - even wireless - until something overwrites it again
(e.g. the well-known pulsing-after-standby behavior).

## If it starts pulsing again

1. Connect the keyboard via USB cable
2. In this folder:
   ```
   .\.venv\Scripts\python.exe set_rgb.py
   ```
3. Unplug the cable again and keep working wireless as usual

That's it. No autostart, no background service needed - the script only
runs when you manually start it as needed.

## One-time setup (only if .venv is missing or broken)

```
powershell -ExecutionPolicy Bypass -File .\setup_venv.ps1
```

This creates `.venv`, installs the `hid` library, and automatically
downloads the required `hidapi.dll`.

## Setting your own color/brightness

At the top of `set_rgb.py`:

```python
RGB_HUE = 43          # 0-255 (QMK hue wheel, not 0-360°). ~43 = Yellow
RGB_SAT = 255          # 0-255, saturation
RGB_BRIGHTNESS = 255   # 0-255, brightness
```

Change the values and run the script again (via cable) - no waiting time,
the effect is immediately visible on the keyboard.

## Troubleshooting

**"Unable to load ... hidapi.dll"**
Native DLL is missing. Run `setup_venv.ps1` again, or load it manually:
https://github.com/libusb/hidapi/releases → `hidapi-win.zip` → copy the
file `x64\hidapi.dll` into this folder.

**"Could not write to any Keychron HID interface" / only "Keychron Link" visible**
The keyboard is only connected wirelessly (2.4G dongle or Bluetooth). Raw
HID only works over cable on the Max series. Connect the cable and run
the script again.

**"Could not write" despite being connected via cable**
- Close the VIA app or Keychron Launcher (browser tab) - they block
  exclusive access to Raw HID
- As a last resort, open PowerShell as administrator and try again

**pip install errors / "file is being used by another process"**
Usually OneDrive sync or antivirus locking `.venv` files. Pause OneDrive
briefly or move the folder out of OneDrive (e.g. to
`C:\Tools\KeychronRGB`), then run `setup_venv.ps1` again.
