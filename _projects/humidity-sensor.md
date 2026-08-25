---
title: "WiFi Soil Moisture Sensor"
summary: "A self-hosted soil moisture and temperature sensor on an Arduino Nano RP2040 Connect — web UI, guided calibration, and WiFi setup with no app or cloud account."
repo: https://github.com/Gharlyk/Humidity-Sensor
---

A capacitive soil moisture sensor and onboard temperature reading, served over a small web page from the device itself. No app, no cloud account, no phone pairing — on first boot it starts its own access point with a form for your WiFi credentials, and after that it's reachable from any browser on your network.

The firmware runs everything as non-blocking state machines from a single `loop()`, so the web UI stays responsive even while a moisture reading is averaging 300 samples over 30 seconds. A guided two-point calibration flow (dry air, then water) lets you tune it to your specific sensor and soil rather than relying on generic thresholds, and the results persist across reboots. Hardware-independent logic is pulled into a small library and unit-tested on the PC, separate from the firmware.

Two writeups cover the build: [getting it onto PlatformIO and chasing four bugs that only appeared on real hardware]({% post_url 2026-08-17-self-hosted-wifi-soil-sensor-rp2040 %}), and [the flash-persistence bug that a full test suite and a code review both missed]({% post_url 2026-08-25-mbed-kvstore-default-broke-flash-persistence %}).
