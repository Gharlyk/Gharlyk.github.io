---
layout: post
title: "Adding Shelly devices to a house without the app: hotspot, WiFi, MQTT and names from one script"
date: 2026-09-17
tags: [shelly, loxone, mqtt, python, raspberry-pi, home-automation]
project_repo: https://github.com/Gharlyk/shelly-for-loxone
excerpt: "A new Shelly boots as a WiFi hotspot; from there it is a dozen settings per device before Loxone can use it. This is the toolkit that does it for a whole house — join the hotspot, put the device on the network, name it, point it at the broker, give it a static IP — with the real console output from the first run at home."
---

## The problem

I run a Loxone Miniserver with a growing number of Shelly devices — plugs, relays, a dimmer, a temperature sensor — talking to it over MQTT through a small Mosquitto broker on a Raspberry Pi. How the Loxone side works, and the traps on the way, is [its own post]({% post_url 2026-09-17-shelly-loxone-mqtt-integration %}). This one is about the part that comes before: getting a new Shelly from the box onto the network with the right settings.

Done by hand, every device is the same ritual. It boots as an open WiFi hotspot; you join that hotspot with your phone, type the house WiFi password, find the device again on the network, give it a name, name its channels, point it at the broker with the right topic, switch on two MQTT flags that are off by default, switch the Shelly cloud and Bluetooth off, decide what the relay does after a power cut, and — if you want Loxone to find it reliably — give it a fixed address. Then the next one. Five relays for a new installation, blinds after that, and the same thing again at a second house that has no broker yet.

I wanted this to be a script: run from my Ubuntu laptop on site or from my Windows PC at home, find the new devices, put them on the network, apply what I decided once, and keep a table of what is where. And a second script that turns a fresh Raspberry Pi into the broker of a new house. Both are in [github.com/Gharlyk/shelly-for-loxone](https://github.com/Gharlyk/shelly-for-loxone).

## What I built

Two tools, sharing one config file per house:

**`setup/` — the Shelly toolkit.** `discover` lists the Shelly hotspots around the laptop and the Shellies already on the LAN. `wifi` joins each hotspot, sends the WiFi credentials, finds the device back on the network by its MAC address and writes a row into a markdown table. I fill in the location and the channel labels next to the MAC. `configure` then finds each device by MAC and applies name, labels, timezone, cloud off, Bluetooth off, MQTT broker and topic, switch or blind defaults, and a static IP — only what differs from the device's current state, so it can be re-run any time. `status` shows everything on one screen, MQTT connection included. Blinds get a cover profile and a `calibrate` command.

**`broker/` — the Pi installer.** `pi-broker install` from the laptop: installs an SSH key, copies an installer and the settings to the Pi over the SSH connection itself, runs it with `sudo` — Mosquitto, one user with a password, anonymous access refused, hostname, timezone — and switches the Pi to its static IP last, with a timer that puts it back on DHCP if the laptop cannot reach it afterwards. `check` then logs in to the broker from the laptop three times: with the right password (must work), a wrong one and no login at all (both must be refused). `listen` subscribes to a device's topics and shows what it really publishes.

Everything is Python 3 standard library plus the `ssh` client every Ubuntu and Windows machine already has. Run either tool without arguments and you get a menu that prints the command it runs, so the direct form is easy to learn.

## The first run, step by step

This is the real session at home, with addresses and names changed. The house network is `192.168.10.0/24`, the broker is the Pi at `192.168.10.20`, three Shellies were already in use, and one factory-fresh Plug S was still in its box.

### 0. One config file per house

```ini
# setup/sites/home/shelly.conf  (git-ignored: it holds passwords)
[WiFi]
ssid = house-wifi
password = ********

[Network]
gateway = 192.168.10.1
dns = 192.168.10.1

[StaticIP]                    ; handed to new devices, in order; outside the router's DHCP pool
start = 192.168.10.60
end = 192.168.10.80

[Device]
timezone = Europe/Berlin
cloud = no
bluetooth = no

[MQTT]
enable = yes
server = 192.168.10.20:1883
user = mqtt
password = ********
topic_prefix = shelly/{location}
status_ntf = yes
enable_control = yes

[Switch]
initial_state = restore_last  ; after a power cut
```

`config` reads it back with passwords masked and lists what is missing; it contacts nothing.

### 1. Who is out there

```
$ ./shelly.sh --site home discover
[1/2] WiFi scan for Shelly access points...
      [+] ShellyPlugSG3-A1B2C3D4E5F6
[2/2] LAN scan 192.168.10.0/24 for Shellies already on your WiFi...
      [+] 192.168.10.31  S3PL-00112EU  shellyplugsg3-3c61050a1b2c
      [+] 192.168.10.32  S3SW-001X8EU  shelly1minig3-3c61050a2d3e
      [+] 192.168.10.33  S3DM-0010WW   shelly0110dimg3-3c61050a4f50

Map updated: 1 new AP-mode device(s), 3 new device(s) on the LAN
```

The WiFi scan is `nmcli` on Ubuntu, `netsh` on Windows; every network called `Shelly…` is a device still in setup mode, and the MAC is in the name. The LAN scan is 254 addresses, ports 80 and 443, threaded, about five seconds; every answering device is asked `Shelly.GetDeviceInfo`. Nothing is written to any device.

### 2. Put the new one on the WiFi

From the laptop, standing near the plug:

```
$ ./shelly.sh --site home wifi ShellyPlugSG3-A1B2C3D4E5F6

=== ShellyPlugSG3-A1B2C3D4E5F6 ===
  joined AP 'ShellyPlugSG3-A1B2C3D4E5F6'
  waiting for the device at 192.168.33.1...
  device shellyplugsg3-a1b2c3d4e5f6  model S3PL-00112EU  fw 20260710-101146/2.0.0-g87fbfa4
  [+] WiFi credentials for 'house-wifi' sent; device is joining your network
  [+] device reports IP 192.168.10.147
  reconnecting this computer to your WiFi...
  [+] on your network at 192.168.10.147
```

Two details make this fast. A new Shelly keeps its hotspot open for 15 minutes after power-on, and after it joins your network the hotspot stays up about five minutes more — long enough to ask the device, still over the hotspot, which address it got from your router. So the laptop switches back to the house WiFi and goes straight to the device instead of scanning. The scan exists as a fallback.

### 3. Fill in the map

`wifi` added a row; I type the location and the channel label next to the MAC:

```
| # | MAC          | IP            | Location    | Ch1 Label   | Ch2 Label | Static IP? | Applied | Notes |
|---|--------------|---------------|-------------|-------------|-----------|------------|---------|-------|
| 1 | 3C61050A1B2C | 192.168.10.31 |             |             |           | no         |         | found on LAN 2026-09-16 |
| 2 | 3C61050A2D3E | 192.168.10.32 | Hallway     | light       |           | no         |         | found on LAN 2026-09-16 |
| 3 | 3C61050A4F50 | 192.168.10.33 | Kitchen     | ceiling     |           | no         |         | found on LAN 2026-09-16 |
| 4 | A1B2C3D4E5F6 | 192.168.10.60 | Office      | desk-plug   |           | yes        |         |       |
```

The map is the source of truth: a device is its MAC; IP, name and labels are what I want it to be; `Applied` says the device agrees. Columns are found by header name, so any editor will do.

### 4. Configure

```
$ ./shelly.sh --site home configure 4

=== #4 Office ===
  reached at 192.168.10.147: shellyplugsg3-a1b2c3d4e5f6 fw 20260710-101146/2.0.0-g87fbfa4
  [+] device name = Office
  [+] switch:0 name = desk-plug
  [+] timezone: {"location": {"tz": "Europe/Berlin"}}
  [+] cloud: {"enable": false}
  [+] bluetooth: {"rpc": {"enable": false}}
  [+] mqtt: {"enable": true, "server": "192.168.10.20:1883", "user": "mqtt", "topic_prefix": "shelly/office", "status_ntf": true, "enable_control": true, "pass": "***"}
  [+] switch:0 defaults: {"initial_state": "restore_last"}
  rebooting the device to apply (MQTT/Bluetooth changes need it)...
  [+] back at 192.168.10.147
  [+] static IP 192.168.10.60 sent; waiting for the device to come back...
  [+] device answers at 192.168.10.60
  [+] MQTT connected to the broker

1/1 device(s) configured.
```

One reboot, because MQTT changed; the static IP last, because it is the change that moves the device. Run it again and every line says `[=] … already set` and nothing is sent.

### 5. Check

```
$ ./shelly.sh --site home status 4

--- 192.168.10.60 ---
  shellyplugsg3-a1b2c3d4e5f6  model S3PL-00112EU  gen 3  fw 20260710-101146/2.0.0-g87fbfa4
  name: Office   MAC A1B2C3D4E5F6   auth: off   provision: complete
  wifi: house-wifi got ip rssi -40 dBm   ip 192.168.10.60 (static)
  tz: Europe/Berlin   cloud: off   bluetooth: off   mqtt: 192.168.10.20:1883 prefix shelly/office connected
  switch:0 'desk-plug': OFF  0.0 W  236.3 V  0.00 A  total 14 Wh   [- / restore_last]
```

And from the broker's side, with the other tool:

```
$ cd ../broker && ./pi-broker.sh --site home listen shelly/office 10
  shelly/office/online  true
  shelly/office/status/switch:0  {"id":0,"source":"init","output":false,"apower":0.0,"voltage":236.3,"current":0.0,"aenergy":{"total":14.0,...},"temperature":{"tC":38.1,...}}
  shelly/office/status/mqtt  {"connected":true}
  ...
```

That is the moment the Loxone side starts: subscribe to `shelly/office/status/switch:0`, publish `on`/`off` to `shelly/office/command/switch:0`.

## How it works

**One API for every Gen2+ device.** Everything from the "Plus" generation onwards speaks the same JSON-RPC over HTTP: `POST /rpc` with `{"method": "WiFi.SetConfig", "params": {...}}`, and likewise `Sys.SetConfig` for the name, `Switch.SetConfig` / `Cover.SetConfig` / `Light.SetConfig` for the channels, `MQTT.SetConfig`, `Cloud.SetConfig`, `BLE.SetConfig`. That makes the toolkit generic for free: labels go to whatever channels the device reports — `switch:0…3` on relays, `cover:0` on a 2PM driving a blind, `light:0` on a dimmer. Devices shipped with firmware 2.0 redirect HTTP to HTTPS with a self-signed certificate; the RPC helper follows that quietly, otherwise every POST would silently turn into an empty GET.

**Only differences are sent.** `configure` reads `Shelly.GetConfig` first and sends the keys that differ. Re-running is harmless, one reboot happens only when MQTT or Bluetooth actually changed, and the MQTT password — which cannot be read back — is only sent alongside another change or when the device reports it is not connected.

**Empty means "leave it alone".** The three devices already in use had MQTT prefixes that Loxone listens to, and a `configure all` with the standard `shelly/{location}` prefix would have renamed them and broken a fan. So: an empty `topic_prefix` keeps the device's prefix, an empty `enable` leaves MQTT untouched, a missing section is skipped entirely. On a house with running devices, `configure <#>` one at a time.

**Blinds.** A 2PM can run as two relays or as one cover. A `Profile` column in the map switches it (`Shelly.SetProfile`, the device reboots), a `[Cover]` section in the conf gives every blind the same input mode and safety timeouts, and `calibrate <#>` runs the blind fully open and closed once so that positions in percent work.

**The broker installer never puts the password on a command line.** The settings file is built in memory on the laptop and streamed to the Pi as a tar archive on the SSH connection's stdin, mode 600, and shredded by the installer when it exits. The installer itself is idempotent: hostname, timezone, apt, a `conf.d/loxone.conf` with the listener and `allow_anonymous false`, `mosquitto_passwd`, service, a local self-test, and the network change last because it is the one step that cuts the connection — with a `systemd-run` timer that reverts to DHCP after three minutes unless the laptop reconnects and cancels it.

**Verification is done from both sides.** On the Pi: service, port, users, a publish/subscribe round trip. From the laptop: a hand-written MQTT 3.1.1 CONNECT packet (a few dozen bytes) and the CONNACK return code — accepted, bad credentials, not authorised.

## What was hard

**Existing devices.** A tool that "configures every Shelly" is dangerous on a house where some already work. Reading the real devices before writing anything, and making empty settings mean "keep", is what turned this from a new-house tool into one I can run at home.

**A static IP inside the DHCP pool.** One plug kept the address the router had given it and I made that address static. The device is fine with that; the router does not know, and could hand the same address to someone else one day. A DHCP reservation for the device's MAC is the second half of a static IP — or a range outside the pool, which is what `[StaticIP]` is for.

**Line endings.** On Windows, Python's `Path.write_text` writes CRLF. The installer's first run on a real Debian (in a Docker container) failed with `$'\r': command not found` — every script in the repo had been silently converted, including the one meant to run on the Ubuntu laptop. All writers now pass `newline="\n"` and a `.gitattributes` enforces LF.

**Testing against real hardware, carefully.** Everything runs first against a fake Shelly on `127.0.0.1` (`setup/tests/`, in the repo) and a fake broker, with an empty static-IP range so that no verification step can wander onto a real address. On the real network I ran one command at a time and read the output before the next one. That is how the topic-prefix problem was caught before it did damage.

**Things the fake device got wrong and only a real one caught:** `Switch.SetConfig` needs `id` beside `config`, not inside it; the Bluetooth config on firmware 2.0 is only `{"rpc": {"enable": …}}`; a factory device reports `bluetooth: on` during its provisioning window and switches it off by itself when the window closes.

## Where it stands

Verified on real hardware at home: three Gen3 devices already on the network (Plug S, 1 Mini, Dimmer 0/1-10V) discovered and read, two of them renamed; one factory-fresh Plug S Gen3 on firmware 2.0.0 provisioned from its hotspot via the Ubuntu laptop, configured with MQTT to the existing broker, then given a static IP, and seen from the broker's side with `listen`.

Not yet on real hardware: the broker installer on an actual Pi (it has run against real Mosquitto 2.0.11 in a Debian container), the 2PM Gen4, and the blind profile with `calibrate`.

## Try it yourself

```
git clone https://github.com/Gharlyk/shelly-for-loxone
cp setup/shelly.conf.example setup/sites/<site>/shelly.conf     # WiFi, network, broker
cd broker && ./pi-broker.sh --site <site> install               # 1. the Pi becomes the broker
cd setup  && ./shelly.sh --site <site> discover                 # 2. who is out there
             ./shelly.sh --site <site> wifi all                 #    put them on the WiFi
             <edit shelly-map.md: Location, Ch1 Label, Ch2 Label>
             ./shelly.sh --site <site> configure all            #    names, MQTT, static IP
             ./shelly.sh --site <site> status all
```

Windows: `.\pi-broker.ps1` / `.\shelly.ps1` with the same arguments. You need Python 3, the OpenSSH client, and a laptop with WiFi for the hotspot step. Nothing uses the Shelly cloud. The top-level README walks through the steps with every command; `setup/README.md` and `broker/README.md` have the details.

If you already have a broker and running devices: leave `[MQTT] enable` and `topic_prefix` empty in your `shelly.conf`, run `discover` and `status all` (read-only), and only then `configure` the devices you mean to.

## What's next

The Pi at the second house is the next real test, then the 2PM relays and the blinds — that is where the cover profile and `calibrate` earn their keep. And a stronger password on the home broker, now that changing it on every device is one command.

---

*Code: [github.com/Gharlyk/shelly-for-loxone](https://github.com/Gharlyk/shelly-for-loxone)*
