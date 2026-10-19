---
title: "TuyaBeGone: a free firmware for a cheap LSC camera, without the vendor libraries"
date: 2026-10-19 15:00:00 +0200
day: "2026-10-19"
short_title: "TuyaBeGone: free firmware for a cheap LSC camera"
description: "Replacing the cloud firmware of an LSC (HT-IPC280, Anyka AK3918AV130) camera with a Buildroot system: video and audio through the kernel drivers only (vendor libraries traced with LD_PRELOAD and rewritten), RTSP and ONVIF with motion events, AAC audio for UniFi Protect, a two-stage SD-card install that keeps the factory sensor calibration, and setup gated by the physical reset button."
tags: [camera, anyka, buildroot, onvif, rtsp, reverse-engineering, unifi-protect, embedded-linux]
---

**Short version:** a cheap indoor camera sold under the LSC brand
(model 3215672, internally HT-IPC280) only works with a Tuya cloud app.
We replaced its firmware with a small Buildroot system. It streams
H.264 and audio over RTSP and ONVIF (profile S, with motion events),
works in UniFi Protect, installs from an SD card without opening the
camera, and does not use any of the vendor's binary user-space
libraries.

## The hardware

- SoC: Anyka AK3918AV130 (ARM926EJ-S, ARMv5), 8 MB of SPI flash.
- Sensor: GalaxyCore GC20C3, 1080p, with IR LEDs and an IR-cut filter.
- Wi-Fi: AltoBeam, 2.4 GHz only.
- Microphone, small speaker, one blue LED, one reset button, micro-SD slot.

The budget is tight: the kernel partition holds 1.5 MB, the system
image 5 MB.

## Video and audio without the vendor libraries

The vendor SDK ships GPL kernel modules (ISP, hardware encoder, PCM,
Wi-Fi) and a set of closed user-space libraries on top of them. We keep
the kernel and its modules, and replace the libraries with our own
small C library, `libtbgav`.

To learn what the closed libraries do, we wrote `iotrace`, an
`LD_PRELOAD` shim. It logs every `open`, `ioctl` (with the argument
before and after), `mmap`, `read` and `write` on `/dev/*`. Running the
vendor demo under it gives the exact sequence each driver expects.
`libtbgav` replays that sequence: V4L2 capture of two NV12 channels
(1080p and 360p), hardware H.264 and JPEG encoding, microphone capture
and speaker playback.

Two driver traps cost us some time. Both the capture and the playback
paths must be served **one period at a time**: a `read()` or `write()`
larger than one period (2048 bytes) blocks forever, because the driver
stops waking the caller once its buffer is full. And every capture
channel must be drained continuously, or the whole pipeline stalls.

Day/night switching (IR-cut filter and IR LEDs) was handled by a closed
"software photosensor" in the vendor library. We read its principle in
Ghidra and rewrote it from scratch, without copying any code, as a few
pure functions over the ISP's exposure and white-balance statistics. A
small test bench runs the original and ours on the same inputs. A
custom "indoor" setting that went back to day mode later turned out to
make the camera oscillate between day and night at certain light
levels, so we went back to the vendor thresholds.

## Streaming: RTSP, ONVIF, UniFi Protect

The streaming daemon (`avstream`) uses live555 for RTSP and
`onvif-simple-server` for ONVIF discovery, profiles and events.

- **Audio**: the first version sent G.711 µ-law. UniFi Protect
  subscribed to the audio track and dropped it right away. With AAC-LC
  (16 kHz mono, 48 kbit/s, encoded with the small Apache-licensed
  `vo-aacenc`), Protect records the sound. G.711 stays available as an
  option. Over 5 minutes, measured drift was −13 ms for video and 0 ms
  for audio. The lip-sync drift we saw came from Protect's live web
  player, not from the camera.
- **Motion events**: the ISP already computes a 32 × 24 grid of
  activity values for its 3D noise reduction. We compare each block
  with its learned background level. If enough blocks stay above it for
  a few frames, that counts as motion, ending 10 s after the last
  detection and ignored for 5 s around each day/night switch. The result
  is published as ONVIF `MotionAlarm` and `CellMotionDetector` events.
  PullPoint subscriptions work. We have not yet checked them in
  Protect.
- **Restart trap**: live555 does not set `SO_REUSEADDR` on its server
  sockets. After a restart with Protect connected, port 554 stayed in
  `TIME_WAIT` for about a minute and the daemon exited. We now create
  the listening socket ourselves and hand it to live555, so the stream
  comes back in about 3 seconds. A small supervisor restarts the daemon
  with back-off if it ever exits.
- Extras: a JPEG snapshot URL, a mute switch that keeps the audio track
  but sends silence, a speaker API (chime, siren, or any audio file
  decoded by the browser and sent as WAV), and short, soft feedback
  sounds for the button and setup steps. The speaker amplifier starts
  late, so each sound begins with 150 ms of silence.

## Installing from an SD card, in two stages

The factory U-Boot has a recovery feature: if a card with a specific
config file is present at boot, it writes the kernel and the
file systems from the card into flash. U-Boot itself and the config
partition are never rewritten. **An SD card can always recover the
camera.**

The catch is the sensor calibration. Each camera's firmware carries
the vendor's ISP tuning file (colour, exposure, noise). We do not
redistribute it, so the install takes it from the camera itself:

1. **Bootstrap.** U-Boot writes only a small kernel with an embedded
   busybox. After a press on the reset button (nothing happens without
   it), this kernel copies the calibration file into the config
   partition, dumps the rest of the old flash to the card for analysis,
   puts the stage-2 files in place and reboots. One more trap here: with
   an lz4-compressed initramfs, the kernel hung before the console came
   up. Gzip works.
2. **Firmware.** U-Boot writes the full system, which boots.

The downside is that the install overwrites the original kernel before
it can be saved. Going back to the factory firmware needs a full flash
dump made through the serial console **before** installing.

Later updates can also go through the web page. The upload is checked
(magic numbers, sizes, version read by loop-mounting the image), and
the camera then writes it from a static busybox copied to RAM. There
is a single system partition and no automatic rollback: the SD card is
the fallback.

## Setup gated by the physical button

Without a configuration, the camera opens an open Wi-Fi access point
with a captive portal. Anyone nearby could join it. So the setup form
is only given to the device that was waiting **alone** when someone
pressed the reset button. Other devices on the access point get a
warning ("if this is not you, unplug the camera"), and if several were
waiting, nobody gets the form. Saving the settings needs a second press
within 2 minutes. The button counter survives a daemon restart, so a
restart cannot pass for a press.

The production image contains no password. The root password is set
at boot from the access password chosen during setup, and root is
locked if there is none. The snapshot URL and the configuration
pages require it. Telnet is off by default and always asks for a login.

## Status

Video, audio, ONVIF, motion events (via PullPoint), the button-gated
setup and the SD install are tested on the camera. The production
hardening and web updates are built but not yet tested on hardware.
The source is not published yet.

{% include coauthors.html %}
