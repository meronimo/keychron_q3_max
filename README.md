#!/usr/bin/env python3
"""
Keychron Q3 - RGB Boot Enforcer
---------------------------------
Sets the RGB Matrix lighting to a fixed static color/brightness via the
VIA Raw-HID protocol, and saves it to the keyboard's EEPROM so it survives
reboots / USB re-enumeration.

Run this once manually to confirm it works, then register it to run a
few seconds after Windows login (see README.md for the Task Scheduler setup).

Requires: pip install hid --break-system-packages   (or just "pip install hid" on Windows)
"""

import os
import sys
import time

# The 'hid' package needs the native hidapi.dll, which setup_venv.ps1 places
# in this same folder. Make sure Windows' DLL loader looks here too -
# needed because os.add_dll_directory is required on Python 3.8+ for DLLs
# that aren't on PATH.
_SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
if sys.platform == "win32" and hasattr(os, "add_dll_directory"):
    os.add_dll_directory(_SCRIPT_DIR)

import hid

# ---------------------------------------------------------------------------
# CONFIG - edit these to taste
# ---------------------------------------------------------------------------

VENDOR_ID = 0x3434  # Keychron, fixed for all their boards

# Desired lighting state
RGB_EFFECT = 1        # 1 = RGB_MATRIX_SOLID_COLOR (static, no animation)
RGB_HUE = 43          # 0-255. Yellow is roughly hue 40-45 in QMK's 0-255 hue space
RGB_SAT = 255         # 0-255, fully saturated yellow (not pastel)
RGB_BRIGHTNESS = 255  # 0-255, max brightness

# ---------------------------------------------------------------------------
# VIA protocol constants (from QMK's quantum/via.h)
# ---------------------------------------------------------------------------

ID_LIGHTING_SET_VALUE = 0x07
ID_LIGHTING_SAVE = 0x09

CHANNEL_RGB_MATRIX = 3

VALUE_RGB_MATRIX_BRIGHTNESS = 1
VALUE_RGB_MATRIX_EFFECT = 2
VALUE_RGB_MATRIX_EFFECT_SPEED = 3
VALUE_RGB_MATRIX_COLOR = 4  # data = [hue, sat]

REPORT_LENGTH = 32  # VIA raw HID reports are 32 bytes


def find_keychron_devices():
    """Return all HID interfaces exposed by Keychron devices."""
    matches = []
    for d in hid.enumerate():
        if d.get("vendor_id") == VENDOR_ID:
            matches.append(d)
    return matches


def pick_via_interface(devices):
    """
    A Keychron Q3 typically exposes more than one HID interface (keyboard,
    consumer control, and the VIA raw HID interface). The VIA interface is
    usually identified by usage_page 0xFF60 and usage 0x61 (QMK's raw HID
    usage), which is the convention QMK firmware uses.
    """
    via_candidates = [
        d for d in devices
        if d.get("usage_page") == 0xFF60 and d.get("usage") == 0x61
    ]
    if via_candidates:
        return via_candidates
    # Fallback: some platforms/backends don't report usage_page correctly.
    return devices


def is_dongle_only(devices):
    """
    Detect the case where only the 2.4G/Bluetooth dongle ("Keychron Link")
    is visible, with no real VIA raw HID endpoint. Raw HID is not relayed
    over the wireless link, so writes to the dongle's interfaces fail with
    an "invalid function" style error even though the device enumerates.
    """
    product_names = {(d.get("product_string") or "") for d in devices}
    return all("link" in name.lower() for name in product_names if name)


def send_via_command(device_handle, data_bytes):
    """Pad to REPORT_LENGTH, prefix with a 0x00 report-id byte, and send."""
    payload = bytes(data_bytes)
    if len(payload) > REPORT_LENGTH:
        raise ValueError("VIA command longer than 32 bytes")
    padded = payload + bytes(REPORT_LENGTH - len(payload))
    report = bytes([0x00]) + padded  # leading 0x00 = HID report ID (none used)
    device_handle.write(report)


def main():
    devices = find_keychron_devices()
    if not devices:
        print("No Keychron device found. Is the Q3 plugged in via USB?")
        sys.exit(1)

    candidates = pick_via_interface(devices)
    if not candidates:
        print("Found Keychron device(s) but no VIA-capable interface.")
        sys.exit(1)

    last_error = None
    for dev_info in candidates:
        try:
            h = hid.Device(path=dev_info["path"])
        except Exception as e:  # noqa: BLE001
            last_error = e
            continue

        try:
            print(f"Connected: {dev_info.get('product_string')} "
                  f"(PID 0x{dev_info.get('product_id', 0):04x})")

            # 1. Set effect to solid color (no animation)
            send_via_command(h, [ID_LIGHTING_SET_VALUE, CHANNEL_RGB_MATRIX,
                                  VALUE_RGB_MATRIX_EFFECT, RGB_EFFECT])
            time.sleep(0.05)

            # 2. Set hue + saturation
            send_via_command(h, [ID_LIGHTING_SET_VALUE, CHANNEL_RGB_MATRIX,
                                  VALUE_RGB_MATRIX_COLOR, RGB_HUE, RGB_SAT])
            time.sleep(0.05)

            # 3. Set brightness
            send_via_command(h, [ID_LIGHTING_SET_VALUE, CHANNEL_RGB_MATRIX,
                                  VALUE_RGB_MATRIX_BRIGHTNESS, RGB_BRIGHTNESS])
            time.sleep(0.05)

            # 4. Persist to EEPROM so it survives reboot/replug
            send_via_command(h, [ID_LIGHTING_SAVE])
            time.sleep(0.05)

            h.close()
            print("Done. RGB set to static yellow at full brightness and saved.")
            return
        except Exception as e:  # noqa: BLE001
            last_error = e
            try:
                h.close()
            except Exception:
                pass
            continue

    print("Could not write to any Keychron HID interface.")
    if last_error:
        print(f"Last error: {last_error}")

    if is_dongle_only(candidates):
        print()
        print("It looks like the keyboard is only connected wirelessly")
        print("(via the 'Keychron Link' 2.4G/Bluetooth dongle). VIA raw HID")
        print("commands are not relayed over the wireless link - only the")
        print("keyboard, mouse, and media-key HID reports get through.")
        print()
        print("Plug the keyboard in with a USB-C cable and run this script")
        print("again. Once the lighting is saved to the keyboard's EEPROM,")
        print("you can unplug the cable and go back to using it wirelessly -")
        print("the setting persists regardless of connection mode.")
    else:
        print("Try running this script as Administrator, or check README.md "
              "for troubleshooting (e.g. quitting VIA/Keychron Launcher first).")
    sys.exit(1)


if __name__ == "__main__":
    main()
