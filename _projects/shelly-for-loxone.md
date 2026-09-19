---
title: "Shelly for Loxone"
summary: "Scripts that take a boxful of Shelly devices from their setup hotspots to named, MQTT-connected members of a Loxone installation — plus an installer that turns a fresh Raspberry Pi into the house's Mosquitto broker."
repo: https://github.com/Gharlyk/shelly-for-loxone
---

Two command-line tools, Python standard library only, run from an Ubuntu laptop or a Windows PC, sharing one config file per house.

`setup/` handles the devices: `discover` lists the Shelly hotspots nearby and the Shellies already on the LAN, `wifi` joins each hotspot and puts the device on the house network, and `configure` finds every device by its MAC address in a markdown table and applies name, channel labels, timezone, cloud and Bluetooth off, MQTT broker and topic, relay or blind defaults and a static IP — sending only what differs from the device's current state, so it can be re-run at any time. `status` shows the whole house on one screen, MQTT connection included.

`broker/` handles the Pi: one command over SSH installs Mosquitto with a password-protected user and anonymous access refused, sets hostname, timezone and a static IP, and verifies the result from both sides — including three raw MQTT logins from the laptop that must be accepted, refused and refused.

Two writeups: [the provisioning toolkit, with the console output of its first run at home]({% post_url 2026-09-17-shelly-provisioning-toolkit %}), and [the Loxone side — topics, device settings, Command Recognition syntax, and the things that failed silently]({% post_url 2026-09-17-shelly-loxone-mqtt-integration %}).
