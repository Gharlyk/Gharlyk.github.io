---
layout: post
title: "The Mbed KVStore Default That Silently Broke My Flash Persistence"
date: 2026-08-25
tags: [embedded, rp2040, mbed, platformio, debugging]
project_repo: https://github.com/Gharlyk/Humidity-Sensor
excerpt: "My soil sensor's calibration refused to survive a reboot. The code was correct. The tests passed. The bug was 8 kilobytes of flash I didn't know I was limited to — and the documented fix didn't work either."
---

## The problem

My WiFi soil moisture sensor — an Arduino Nano RP2040 Connect with a small self-hosted web UI — has a two-point calibration flow. You put the probe in dry air, hit a button, put it in water, hit another button, and it learns the raw ADC values that correspond to 0% and 100% moisture.

The catch: those values only lived in RAM. Every reboot silently threw them away and reverted to hardcoded defaults, so the readings drifted back to being wrong. The fix seemed trivial — write two integers to flash on save, read them back on boot. The device already stored WiFi credentials in flash using exactly that mechanism, so there was a working pattern to copy.

I wrote it. It built. The unit tests passed. Then I flashed it to the actual board, ran a calibration, power-cycled, and watched it come back up with the default values again.

## What I built

Before getting to the interesting failure, the boring context: this session added calibration persistence, MQTT publishing, a WiFi reconnect watchdog, and a fix for a URL-decoding bug that would have corrupted any WiFi password containing a `%`, `+`, or `&`.

I was reasonably confident in all of it, because I'd been thorough:

- **34 unit tests passing** on the host PC, covering all the hardware-independent logic (moisture math, HTTP parameter parsing, URL decoding, HTML escaping, config validation).
- **`cppcheck` static analysis clean** — zero high or medium findings.
- **A deep multi-agent code review** that turned up 11 confirmed bugs, all of which I fixed. Some were genuinely good catches: two "non-blocking" watchdogs I'd written were actually capable of blocking the main loop for up to 50 and 15 seconds respectively, because I'd trusted my own comments instead of reading the vendored library source. Another was a stored HTML-injection hole in a form field I'd just added.

None of that caught the bug that actually mattered. It wasn't in my code at all.

## How it works — the debugging

The failure mode was specific and strange: **saving worked for some keys and not others.** WiFi credentials saved fine. Calibration values didn't. Same code path, same API, same flash.

The first thing I did was stop guessing and get the actual error code. The storage wrapper was discarding the return value from Mbed's `kv_set()`, so I made it log the real number.

That took two attempts, and the failed one is worth mentioning. I added `Serial.print()` calls — which was wrong for a reason I should have anticipated. The device's debug log is a ring buffer served over HTTP, and raw `Serial.print()` doesn't go through it. Worse, I'd already discovered that **the serial connection is unreliable on this board across a physical unplug/replug**: the monitor process stays alive and silently receives nothing, even when the OS re-enumerates the same `COM3` port. So I'd routed my diagnostic to the one channel that provably didn't work, and "no output" looked identical to "no error."

Switching the diagnostic to the HTTP-visible logger gave me the number immediately:

```
[FlashStore] kv_set failed for key 'calib_dry': error code -2130771701
```

Decoded: `0x80FF010B` → Mbed system error 267 → **`MEDIA_FULL`**. Out of space.

Which made no sense. The board has megabytes of flash and my firmware uses a few kilobytes of it.

The answer was in mbed-os's own source, not its documentation. `kv_config.cpp` contains this:

```c
// Use the last 2 sectors or 14 pages of flash for the TDBStore by default
static const int STORE_SECTORS = 2;
```

**The default key-value store is the last two flash sectors.** On the RP2040 a sector is 4KB, so the entire store is 8KB — and `TDBStore` splits that into two areas for garbage collection, leaving roughly 4KB usable minus per-area bookkeeping. That was the whole budget for every piece of config the device stores.

It also explained the selective failure perfectly. `wifi_ssid` and `wifi_pass` already had space allocated, so *updating* them worked. `calib_dry` and `calib_wet` were brand-new keys that needed *new* space, and there wasn't any.

## The fix that didn't work

Every Mbed document says the same thing: set `storage_tdb_internal.internal_size` in `mbed_app.json`.

That does nothing here, and it took reading the installed framework to understand why. This board builds against a **precompiled `libmbed.a`** shipped inside the PlatformIO package. The storage configuration was baked in when Arduino compiled that archive — `MBED_CONF_STORAGE_TDB_INTERNAL_INTERNAL_SIZE` is already `0` inside it, and nothing you put in a config file will ever be recompiled. The variant directory even ships the `mbed_app.json` Arduino used to build the library, which is a *record* of the build, not an input to yours.

If I'd trusted the documentation and started editing config files, I'd have burned an entire flash-and-test cycle proving nothing.

## The fix that did

Mbed anticipated this. In `kv_config.h`:

> *"To overwrite the default configuration, please overwrite this function."*

`kv_init_storage_config()` is declared `MBED_WEAK`. A weak symbol can be replaced at link time by a strong definition of the same name — so I can define my own version in my own source file, and the linker will prefer mine over the library's.

I verified that was actually true before writing anything, using `nm` on the archive:

```
W kv_init_storage_config      ← weak, overridable
T FlashIAPBlockDevice::FlashIAPBlockDevice(unsigned long, unsigned long)
T mbed::TDBStore::TDBStore(mbed::BlockDevice*)
T mbed::KVMap::attach(char const*, mbed::kvstore_config_t*)
```

Weak symbol confirmed, and every building block I'd need already compiled into the library with headers shipped. So `src/KVStoreConfig.cpp` defines a strong `kv_init_storage_config()` that mirrors upstream's internal-TDBStore setup, but carves out an explicit **64KB** region instead of accepting the 2-sector default — anchored at the end of flash, with the geometry queried at runtime rather than hardcoded.

Then, before spending a hardware cycle, I checked the override had actually won the link. `nm` on the final ELF:

```
10000a9c T kvStoreConfiguredSize()        ← my function
10000aa4 T kv_init_storage_config         ← immediately after it
...
20000a30 b kv_init_storage_config::bd     ← my static locals
20002348 b kv_init_storage_config::tdb
```

The two functions sit adjacent, which means they came from the same object file — mine. And the BSS section contains `kv_init_storage_config::bd` and `::tdb`, static variables that exist only in my implementation; the library's version has entirely different ones. Proof at link time, before touching the board.

On hardware, it worked:

```
[INFO] KVStore size: 64 KB
[INFO] Loaded calibration from flash: dry=2308 wet=1376
```

Those are the values from the calibration run before the power cycle. First time it's ever persisted.

A footnote that fell out of reading the linker script: it declares `FLASH LENGTH = 16*1024*1024`. **The Nano RP2040 Connect has 16MB of flash**, not the 2MB PlatformIO's build output reports. The board manifest is simply wrong. My 64KB store is 0.4% of what's actually there.

## What was hard / what I'd do differently

**Passing tests and clean analysis measured the wrong thing.** The 34 tests, the clean `cppcheck`, and the 11-finding review all examined logic I wrote. This bug lived in a default constant inside a precompiled third-party library. No amount of scrutiny aimed at my own code was going to surface it — only running it on real hardware did. That's not an argument against the tests, which caught real problems; it's a reminder about what a green build actually proves.

**I keep relearning "read the source, not the docs."** This project has a standing lesson from an earlier session: a web search told me this board used a different Arduino core than it does, and I rewrote a working storage layer on that bad information before an actual build corrected me. This time the same trap was set differently — the docs described a fix that's correct for Mbed in general and useless for this particular precompiled setup. Both times, the vendored source had the real answer. Both times it was faster to go read it.

**Verify your diagnostic channel before trusting its silence.** Routing debug output to a serial connection I already knew was unreliable, and then reading "no output" as "no error," cost me a round trip. When you add instrumentation to diagnose something, confirm the instrumentation itself works — otherwise a broken channel is indistinguishable from a passing test.

**Check what you can before each hardware cycle.** Every flash meant putting the board in bootloader mode, uploading, power-cycling, waiting for WiFi, and re-checking. That's slow, so anything verifiable on the host — does it compile, did the weak override win the link, does `nm` show my symbols — is worth doing first. The link-time check turned a "flash it and hope" into a "flash it knowing."

**What I'd redesign:** the whole storage layer silently assumed writes succeed. `saveCalibration()` logged failures but the calibration state machine advanced regardless, so the UI cheerfully displayed "Calibration complete" for values that had never been written. The device knew the write failed and told nobody who mattered. Surfacing storage failures in the web UI, not just a debug log, is the obvious next step.

One cost worth flagging if you try this yourself: resizing the store relocates its areas and master records, so **anything already stored becomes unreadable**. I had to re-enter WiFi credentials once through the setup portal afterward. One-time, but don't do it remotely to a device you can't physically reach.

## Try it yourself

Code is at [github.com/Gharlyk/Humidity-Sensor](https://github.com/Gharlyk/Humidity-Sensor) — build and flash instructions are in the README. The KVStore override is self-contained in [`src/KVStoreConfig.cpp`](https://github.com/Gharlyk/Humidity-Sensor/blob/main/src/KVStoreConfig.cpp) and should transplant to any Arduino-Mbed board where you've hit the same 8KB wall; adjust `KVSTORE_SIZE` and confirm your board's sector size.

If you're debugging something similar, the two numbers worth knowing: `-2130771701` is `MEDIA_FULL` (out of space) and `-2130771705` is `ITEM_NOT_FOUND` (key was never written — benign, and easy to mistake for a failure).

## What's next

Calibration persistence is done and verified. Still on the list: MQTT publishing works in principle and saves its config correctly now, but hasn't been tested against a real broker. The WiFi reconnect watchdog has fired correctly once by accident and never been deliberately tested. And the calibration itself is still two-point against air and water, when the research suggests calibrating the wet endpoint against actual saturated soil would track real moisture more closely.

There's also one loose thread I noted and chose not to chase: on the very first boot after resizing the store, the device appeared to read WiFi credentials successfully but then failed every connection retry. Most likely remnant data from the old 8KB region, which the new 64KB region overlaps at its tail, being partially readable before the store settled. Re-entering credentials fixed it permanently and every boot since has been clean — but I don't have a confirmed explanation, and it would be dishonest to present one.

---

*Code: [github.com/Gharlyk/Humidity-Sensor](https://github.com/Gharlyk/Humidity-Sensor) · Built with [Claude Code](https://claude.com/claude-code)*
