---
name: user_device
description: User's Supernote Nomad hardware and firmware details
type: user
---

User owns a Supernote Nomad (A6X2) running Chauvet 3.26.40 (latest as of April 2026). Research findings in this project are based on documentation up to 3.23.32 — anything beyond that is unconfirmed on their firmware.

## Hardware Profile (verified via ADB on 2026-04-09)

**Device:** Supernote Nomad (also marketed as A5 X2 Manta)
**Serial:** SN078D10008613
**Board:** A6X2

**SoC:** Rockchip RK3566 (rk356x), quad-core ARM Cortex-A55, ARMv8, 48 BogoMIPS/core
**GPU:** Mali
**RAM:** ~3.8 GB physical + ~1.35 GB zram swap

**Display:** E-ink, 300 DPI, 270° hardware rotation
**Input:** Wacom stylus (i2c-3), capacitive touchscreen, physical slider

**OS:** Android 11 (SDK 30), custom Supernote build — Chauvet.E003.2512251001.2157 (built Dec 25, 2025)
**Security patch:** 2024-02-05

**Connectivity:** Wi-Fi (wlan0), Bluetooth, USB-C (MTP + ADB)
**No:** GPS, NFC, SIM/radio
**Storage:** eMMC, dynamic partitions, FUSE, exFAT/NTFS supported
**CPU ABI:** arm64-v8a, armeabi-v7a, armeabi

## USB Display Forwarding

The Nomad has a built-in browser-based screen share feature (Wi-Fi) on port 8080. To use it over USB:

```
adb forward tcp:8080 tcp:8080
```

Then open `http://localhost:8080` in the browser on the Mac. No app or RE needed.
