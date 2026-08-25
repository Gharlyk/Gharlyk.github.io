---
layout: post
title: "Building a Self-Hosted WiFi Soil Sensor on an RP2040 (and Chasing Four Bugs That Only Showed Up on Real Hardware)"
date: 2026-08-17
tags: [embedded, arduino, rp2040, platformio, iot]
project_repo: https://github.com/Gharlyk/Humidity-Sensor
excerpt: "Migrating 20 versions of an Arduino sketch to PlatformIO, making sensor sampling non-blocking, and debugging a string of bugs that only existed once earlier ones were fixed."
---

> **Update (25 Aug 2026):** the "what's next" items below are done — calibration now persists across reboots, and MQTT and WiFi reconnect handling are in. Getting persistence working turned out to be its own story: it was blocked by an 8KB flash limit I didn't know existed, and the documented fix didn't work. That's written up separately in [The Mbed KVStore Default That Silently Broke My Flash Persistence]({% post_url 2026-08-25-mbed-kvstore-default-broke-flash-persistence %}).

## The problem

I had a soil moisture and temperature sensor project — an Arduino Nano RP2040 Connect reading a capacitive moisture sensor and its onboard IMU for temperature, serving both over a small web UI with a WiFi setup portal and a calibration flow. It worked, mostly, but it had grown across 20 incrementally-versioned Arduino IDE sketches (V0 through V1.8.2), had no automated tests, and — most importantly — blocked the entire device for 30 seconds every time it took a moisture reading. During that window the web server couldn't respond to anything.

The plan: consolidate into one real project, make sampling non-blocking, harden the config portal, and eventually get it publishing over MQTT. This writeup covers getting the project onto PlatformIO and through the first non-blocking-sampling milestone — which turned into a much better debugging story than "wrote some code and it worked."

## What I built

The firmware now runs everything — sensor sampling, an LED status indicator, WiFi connection handling, and a small hand-rolled HTTP server — as non-blocking state machines polled from a single `loop()`, using `millis()`-based timing instead of `delay()`. A moisture reading still averages 300 samples over about 30 seconds for accuracy, but the web UI keeps responding the whole time instead of freezing.

Along the way I also migrated the project to PlatformIO (from 20 loose Arduino IDE sketch folders to one real project with `src/`, `lib/`, and `test/`), pulled the hardware-independent logic (moisture-percentage math, HTTP parameter parsing, sample averaging) into a small library that's unit-tested on the PC, and fixed a calibration flow that — it turned out — had never actually worked.

## How it works

The core design decision was splitting logic by whether it touches hardware. Anything that's pure computation — converting a raw ADC reading to a moisture percentage, parsing a query parameter out of an HTTP request, accumulating and averaging a batch of sensor samples — lives in a small library with zero Arduino dependency, and gets unit-tested on the PC via PlatformIO's `native` environment. Everything that actually touches `millis()`, `analogRead()`, or the WiFi hardware stays in the firmware itself, where it can't be unit-tested but is kept as thin as possible.

For non-blocking sampling specifically: instead of a function that loops 300 times calling `analogRead()` then `delay(100ms)` — blocking for 30 seconds straight — there's now a small accumulator class (`MoistureSampler`) that takes one reading at a time, and a `loop()`-polled function that only takes the next reading once enough time has passed. The same accumulator serves both the regular periodic measurement and the calibration flow, tagged with a "purpose" so the result gets routed to the right place once it's done.

WiFi credentials are stored using the RP2040's Mbed-OS `KVStore` — flash memory in a region separate from the program code itself, which turned out to matter (see below).

## What was hard / what I'd do differently

Almost every bug in this project only became visible once an earlier bug was fixed, in a strict dependency chain:

**The board/core research trap.** Early on, web research suggested this board's PlatformIO platform builds against a different core (`earlephilhower`/arduino-pico) than what the original sketches assumed (Arduino's Mbed OS core), which would have broken the flash-storage API the WiFi credential code relied on. I acted on that and rewrote credential storage before ever actually trying a build — and the research was wrong. A real `pio run` proved the board only ships the Mbed core; forcing the other one just broke the build outright. Lesson: for board/toolchain questions, running the actual build is worth more than search results, and should come *before* writing code around an assumption, not after.

**No compiler on Windows.** Getting the PC-side unit tests running needed a real C/C++ compiler, and there wasn't one on Windows at all. Ended up running PlatformIO from WSL (which already had `gcc`) against the same Windows-side project files, in a Python venv to work around Ubuntu 24.04's externally-managed-environment restrictions.

**Flash persists across reflashing.** The first real upload connected straight to my home WiFi instead of starting the expected first-boot setup portal — turned out the flash region backing `KVStore` isn't touched by a normal firmware reflash, so credentials saved by the *original* pre-migration sketch, tested on the same physical board, were still there. Not a bug, but a genuine surprise worth knowing about if you're reusing a board across firmware rewrites.

**"Connected" doesn't mean reachable.** That same connection turned out to be a dead end — the device wasn't actually visible on the network, despite the boot log claiming success, and there was no code anywhere that noticed or recovered from a dropped WiFi link. The device was alive (sensor readings kept logging on schedule) with a completely dead network connection. Still an open item.

**Auto-refresh reset loop.** Once non-blocking sampling was working and the previously broken calibration flow could finally be reached, it immediately got stuck on "Settling before DRY sampling... please wait" forever. The settling page's auto-refresh reloads the browser's *current* URL — which still had the "start calibration" query string on it — and that route unconditionally reset the settle timer on every hit, so it reset itself faster than the 5-second timer could ever elapse. Fixed by only allowing that reset from the state a user is actually meant to trigger it from.

**A pin type that isn't an int.** A small one: this board's onboard LEDs aren't addressed by plain pin numbers — they're objects representing pins on a *separate* WiFi co-processor, and that type deliberately blocks being treated as an ordinary integer, specifically to stop code like mine from doing exactly what I first tried to do. Another thing only a real build caught.

If I did this again, I'd get a build running against real hardware earlier and lean on it more, instead of trusting research or reasoning about behavior I could just go test directly — a theme that kept repeating across very different parts of this project.

## Try it yourself

Repo, build/upload instructions, and project structure are in the [README](https://github.com/Gharlyk/Humidity-Sensor#readme).

## What's next

Calibration values currently reset to hardcoded defaults on every reboot instead of being saved — that's next, along with adding MQTT broker settings to the config portal and actually wiring up MQTT publishing (the dependency's already in the build, unused). WiFi reconnect handling — so the device can recover from a dropped connection without a manual power cycle — is also still an open gap.

---

*Code: [github.com/Gharlyk/Humidity-Sensor](https://github.com/Gharlyk/Humidity-Sensor) · Built with [Claude Code](https://claude.com/claude-code)*
