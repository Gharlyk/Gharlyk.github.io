---
layout: post
title: "Shelly devices in Loxone over MQTT: how to do it"
date: 2026-09-17
tags: [loxone, shelly, mqtt, mosquitto, home-automation]
project_repo: https://github.com/Gharlyk/shelly-for-loxone
excerpt: "A Shelly plug, a dimmer and a temperature sensor wired into a Loxone Miniserver through a Mosquitto broker on a Pi Zero — the broker setup, the exact device settings, the topics, Loxone's Command Recognition syntax, and a checklist for when nothing arrives."
---

## Why MQTT

Loxone's own components are excellent but expensive. If you want to save some money on a smart plug, a fan or a pump driven by a dimmer, or a temperature and humidity sensor in a room where you don't want to pull a cable, a Shelly does the job for a fraction of the price — and since the "Plus" generation every Shelly speaks the same API, so what works for one works for all of them.

There are two ways to connect a Shelly to a Loxone Miniserver: webhooks or MQTT. I chose MQTT for these reasons:

| | Webhook | MQTT |
|---|---|---|
| Broker needed | no | yes |
| Direction | device → Miniserver only | both |
| Send commands back | not directly | yes |
| Setup | simple | a bit more, much more powerful |

The Miniserver Gen2 has a native MQTT add-on, so once a broker is running you get state *and* control through one place, with one consistent naming scheme for every device. This post walks through the whole thing for three different kinds of device — a Plug S, a Dimmer 0/1-10V and an H&T sensor: the broker, the device settings, the topics, and the Loxone side. The scripts that now do the device part automatically are [a separate post]({% post_url 2026-09-17-shelly-provisioning-toolkit %}); they live in the [repo](https://github.com/Gharlyk/shelly-for-loxone).

Addresses and names below are made up; the network is `192.168.10.0/24`, the broker is `192.168.10.20`, the broker user is `mqtt`.

## MQTT in two minutes

If you have never used MQTT: it is a tiny publish/subscribe protocol. One machine on your network runs a **broker** — think of it as a post office. Every device **publishes** messages to named mailboxes called **topics** (`shelly/office/status/switch:0`, for example), and anyone interested **subscribes** to the topics they care about; the broker delivers. Devices never talk to each other directly, and neither Loxone nor the Shellies need to know each other's address — they only need the broker's.

So the setup has three parts, and the order matters:

1. a broker (Mosquitto on a Raspberry Pi),
2. every Shelly pointed at that broker, each with its own topic prefix,
3. Loxone subscribed to the topics it needs, and publishing commands back.

## Part 1 — the broker

**Hardware.** A Raspberry Pi — a Pi Zero W in my case, powered from a plain USB port. That's all the broker needs; it does almost nothing all day.

**Software.** Mosquitto, the standard open-source broker. On Raspberry Pi OS:

```bash
sudo apt install mosquitto mosquitto-clients
```

`mosquitto-clients` gives you `mosquitto_sub` and `mosquitto_pub`, which you will use constantly to see what is going on.

**Configuration.** One listener on port 1883, one user with a password, anonymous connections refused, and no TLS because everything stays on the home LAN. Create a config file — I called it `loxone.conf`:

```
# /etc/mosquitto/conf.d/loxone.conf
listener 1883
allow_anonymous false
password_file /etc/mosquitto/passwd
```

Then create the user (`mqtt` here — you'll be asked for the password) and start the service so it comes back after every reboot:

```bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd mqtt
sudo systemctl enable --now mosquitto
```

**A fixed address.** Every Shelly and the Miniserver will carry the broker's IP in their configuration, so the Pi must not change address. Two simple ways, pick one:

- *On the router* (simplest): look up the Pi's MAC address in the router's DHCP client list and add a **DHCP reservation** for it. The Pi keeps asking for an address the normal way and always gets the same one. Nothing to change on the Pi.
- *On the Pi*: `sudo nmtui` → *Edit a connection* → pick your WiFi (or Ethernet) connection → set *IPv4 CONFIGURATION* to `Manual` and enter address `192.168.10.20/24`, gateway `192.168.10.1`, DNS `192.168.10.1` → OK, then reboot. If you do this, choose an address outside the router's DHCP range so it can never be handed out twice.

**Check it works** before going any further, from any computer on the LAN with the Mosquitto clients installed (`sudo apt install mosquitto-clients` on Ubuntu). In one terminal, subscribe to everything:

```bash
mosquitto_sub -h 192.168.10.20 -u mqtt -P '********' -t '#' -v
```

In another, publish something:

```bash
mosquitto_pub -h 192.168.10.20 -u mqtt -P '********' -t test -m hello
```

`test hello` appearing in the first terminal means the broker, the user and the network are all fine.

> **Tip.** The repo's `broker/` tool does all of the above over SSH from your laptop, including the static IP and the check. By hand it's ten minutes, and it's worth doing once by hand to know what the tool does.

## Part 2 — topics, the one concept to understand

Every Shelly puts all its MQTT traffic under a **topic prefix**. The default is the device ID (something like `shellyplugsg3-a1b2c3d4e5f6`), but you can set it to anything, and a readable scheme such as `shelly/<room>` makes the Loxone side much easier to maintain. Under that prefix, four families of topics matter:

| Topic | Purpose |
|---|---|
| `shelly/office/online` | presence: `true` / `false` — the broker publishes `false` itself when the device drops off (a "last will" message) |
| `shelly/office/events/rpc` | a push notification the moment something changes |
| `shelly/office/status/<component>` | full status of one component (`switch:0`, `light:0`, `temperature:0`…) — **only if `status_ntf` is on** |
| `shelly/office/command/<component>` | where you publish commands — **only if `enable_control` is on** |

Those two flags are the whole trick. Both are **off by default**, and a device with them off connects to the broker perfectly well and looks healthy — it just never publishes a full status and never listens for commands.

## Part 3 — the devices

### Shelly Plug S

Every Gen2+ Shelly has a small web page at `http://<device-ip>/`. Under *Settings → MQTT* you'll find exactly the fields below; tick *Enable*, fill in the server, user and password, set the prefix, and tick the two checkboxes for MQTT control and generic status updates. If you prefer to do it from a terminal, the same settings as one HTTP call:

```bash
curl -X POST http://<device-ip>/rpc -d '{"id": 1, "method": "MQTT.SetConfig", "params": {"config": {
  "enable": true,
  "server": "192.168.10.20:1883",
  "user": "mqtt",
  "pass": "********",
  "topic_prefix": "shelly/office",
  "rpc_ntf": true,
  "status_ntf": true,
  "enable_control": true
}}}'
```

The device reboots to apply. From then on it uses these topics:

| Purpose | Topic | Payload |
|---|---|---|
| switch on / off | `shelly/office/command/switch:0` | `on` / `off` / `toggle` |
| state and power, live | `shelly/office/events/rpc` | JSON notification |
| full status incl. errors | `shelly/office/status/switch:0` | `{"output":true,"apower":42.3,"voltage":233.1,"aenergy":{"total":1234.5},"errors":[…]}` |
| presence | `shelly/office/online` | `true` / `false` |

Now watch it, and switch it, from your laptop — before Loxone is involved at all:

```bash
mosquitto_sub -h 192.168.10.20 -u mqtt -P '********' -t 'shelly/office/#' -v
mosquitto_pub -h 192.168.10.20 -u mqtt -P '********' -t shelly/office/command/switch:0 -m on
```

The first command shows `online true` and a `status/switch:0` line right after the reboot; the second makes the relay click and a new status line appear with `"output":true`. If both happen, the device side is done.

One thing to know about errors: the interesting ones (`overtemp`, `overpower`, `overvoltage`, `undervoltage`) come in the `errors` array of `status/switch:0`. There is also an `error/switch:0` topic, but it only reports malformed MQTT commands, not device health.

### Shelly Dimmer 0/1-10V

Same MQTT settings, with `topic_prefix` = `shelly/kitchen`, and **firmware 1.6.0 or newer** — MQTT control of the light component simply doesn't exist before that, so update the device first (*Settings → Firmware*). The dimmer's component is `light:0` instead of `switch:0`, and the commands get two extra fields for brightness and fade time:

```bash
# on at 75 %
mosquitto_pub … -t shelly/kitchen/command/light:0 -m "set,true,75"
# off with a 2-second fade
mosquitto_pub … -t shelly/kitchen/command/light:0 -m "set,false,,2"
# toggle
mosquitto_pub … -t shelly/kitchen/command/light:0 -m "toggle"
```

If you'd rather send JSON than the compact `set,true,75` form, every device also listens on `<prefix>/rpc` and accepts any API method:

```bash
mosquitto_pub … -t shelly/kitchen/rpc \
  -m '{"id":1,"src":"loxone","method":"Light.Set","params":{"id":0,"on":true,"brightness":75}}'
```

This is a separate channel from MQTT control and works even with `enable_control` off.

### Shelly H&T (temperature and humidity)

This one is battery powered, which changes everything: no commands, no permanent connection, and it publishes as rarely as it can get away with. Same MQTT settings with `topic_prefix` = `shelly/bedroom` and `enable_control` = `false` — there is nothing to control. The values arrive on:

| Value | Topic | Field |
|---|---|---|
| temperature | `shelly/bedroom/status/temperature:0` | `tC` |
| humidity | `shelly/bedroom/status/humidity:0` | `rh` |
| battery | `shelly/bedroom/status/devicepower:0` | `percent`, `V` |
| presence | `shelly/bedroom/online` | `true` / `false` |

How often depends on how it is powered. On battery it sleeps and only wakes when the temperature moves by more than `report_thr_C` (default 0.5 °C) or the humidity by more than `report_thr` (default 2 %), plus one report every 2 hours no matter what (fixed, not configurable). On USB power it wakes every 5–6 minutes and reports whether anything changed or not. I checked this rather than trusting the documentation, by logging the traffic with timestamps:

```bash
mosquitto_sub -h 192.168.10.20 -u mqtt -P '********' -t 'shelly/bedroom/#' -v \
  | while read -r line; do echo "$(date '+%F %T') $line"; done | tee -a ht.log
```

On USB it came in on a steady ~6-minute cycle. To watch topics from a phone, any MQTT client app pointed at the broker with the same login does the job; subscribe with QoS 1, which is what the device publishes at.

## Part 4 — the Loxone side

In Loxone Config, go to the Miniserver's MQTT add-on and enter the broker: `192.168.10.20`, port `1883`, user `mqtt`, the password. From there, every value you want to use is two building blocks in a row:

```
[Subscription]  →  [Command Recognition]  →  [Virtual Status / Formula / …]
   (raw JSON)        (extracts one number)
```

The **Subscription** is just a topic — `shelly/office/status/switch:0` — and it hands you the whole JSON payload as text. **Command Recognition** pulls one number out of it. It does *not* use JSONPath (that was my first assumption); it uses Loxone's own anchors: `\i…\i` finds an exact piece of text in the payload, and `\v` takes the number that follows it. So:

```
\i"tC":\i\v          → temperature
\i"apower":\i\v      → power in watts
\i"total":\i\v       → cumulated energy in Wh
\i"percent":\i\v     → battery
```

**Booleans** need two recognitions with fixed values, because `\v` only understands numbers:

```
\i"output":true\i    → fixed value 1
\i"output":false\i   → fixed value 0
```

**Sending commands back.** The MQTT Publish block has a topic and sends whatever value you feed it, with no conversion. The Shelly expects the text `on` / `off`, not 0 / 1, so put a Status block in between to translate:

```
[Switch 0/1] → [Status: 0 → "off", 1 → "on"] → [MQTT Publish  shelly/office/command/switch:0]
```

For the dimmer, the Status block builds `set,true,<value>` from the dimmer's percentage; for a scene, `toggle` is enough.

My rule of thumb: always confirm the raw flow with `mosquitto_sub` / `mosquitto_pub` first, then wire it into Loxone. Then each side can be checked on its own when something is off.

## If something doesn't work

Things to check, in the order that found my problems fastest:

1. **Is `enable_control` on?** Off by default. Without it the device never subscribes to `command/…` and silently ignores everything you publish there.
2. **Is `status_ntf` on?** Also off by default. Without it the `status/…` topics — power, energy, errors, temperature — never appear; you only get the terse `events/rpc` notifications.
3. **Firmware.** The dimmer needs 1.6.0+ for MQTT control of `light:0`. Older firmware accepts the setting and ignores the commands, with no error anywhere.
4. **Use the broker's IP, never a hostname.** mDNS (`.local`) resolution on these devices is unreliable and cost me a whole evening of "connection failed" on one device.
5. **Watch from the broker's side.** `mosquitto_sub -t '#' -v` shows every message that reaches the broker, and `sudo tail -f /var/log/mosquitto/mosquitto.log` shows connections and refused logins.
6. **Ask the device what it thinks.** `curl http://<device-ip>/rpc/MQTT.GetConfig` for the settings, `curl -X POST -d '{"id":1,"method":"MQTT.GetStatus"}' http://<device-ip>/rpc` for `"connected": true/false`.
7. **Can the device reach the broker at all?** There is no ping on a Shelly, but `curl 'http://<device-ip>/rpc/HTTP.GET?url=http://192.168.10.20'` makes it try. On the broker, `arp -n | grep <device-ip>` showing `(incomplete)` means the device never even answered ARP — a WiFi or broadcast problem, not an MQTT one.
8. **A device fine over HTTP can still fail over MQTT.** HTTP is one request at a time and forgives a weak WiFi signal; MQTT holds a permanent connection and drops much sooner under the same conditions. Check the RSSI on the device page.
9. **Battery devices behave differently on USB.** The same H&T reports every ~6 minutes on USB and as rarely as every 2 hours on battery. Not a fault.
10. **Factory reset** is sometimes the quickest way out of a state that config changes can't untangle.

## Closing

What started as "how do I switch a plug from Loxone" turned into a small MQTT setup that now carries every Shelly in the house, with one naming scheme and one place to look when something is off. The device side of it — hotspot, WiFi, name, MQTT settings, static IP, for a whole house at once — is now a script; that is the other post.

---

*Code: [github.com/Gharlyk/shelly-for-loxone](https://github.com/Gharlyk/shelly-for-loxone)*
