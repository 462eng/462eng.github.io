---
title: "TuyaBeGone 1.0: when the factory U-Boot only erases 4 KiB"
date: 2026-10-23 20:30:00 +0200
day: "2026-10-23"
short_title: "TuyaBeGone 1.0: a relay for a lazy U-Boot"
description: "The free firmware for the LSC (HT-IPC280, Anyka AK3918AV130) camera reaches 1.0. On new cameras, the factory U-Boot (V4.0.27, November 2025) erases only the first 4 KiB sector of each flash area, so the SD card install wrote over old data. Fix: U-Boot now writes a relay of under 4 KiB that finds fatload and bootm in U-Boot's command table and starts the installer from the card; the installer erases and writes the flash from Linux. Plus verified erases over UART and a relaxed cross-camera factory restore."
tags: [camera, anyka, u-boot, embedded-linux, firmware-update, flash]
---

**Short version:** the "new camera" test announced
[yesterday]({% post_url 2026-10-22-tuyabegone-test-bench-installer-signed-updates %})
failed in an instructive way. Its factory U-Boot is a newer build than
the one on the bench camera, and it **erases only the first 4 KiB
sector** of each flash area before writing. The SD card install has
been reworked around that: U-Boot now writes only a tiny relay, and
the real installer runs from the card. Validated on both U-Boot
versions, this is **TuyaBeGone 1.0**.

## Recap

TuyaBeGone replaces the Tuya cloud firmware of the LSC 3215672
(HT-IPC280, Anyka AK3918AV130 SoC, 8 MB SPI flash) with a local one:
RTSP, ONVIF profile S, UniFi Protect adoption, motion events, signed
updates. The
[first post]({% post_url 2026-10-19-tuyabegone-free-firmware-lsc-camera %})
covers the reverse engineering, the second one the installer and the
way back.

The install relies on a recovery mechanism of the factory U-Boot. At
every boot, if the SD card holds `ht_recover_fw.conf`, U-Boot writes
the files next to it (kernel, system, application) into the flash. It
cannot boot from the card, only write. Our first stage was therefore a
kernel written over the factory one, which then checks the card and
installs the rest.

## What went wrong on the new camera

On a brand-new camera, the SD card install went as usual at first:
U-Boot wrote our first stage and booted it. Then it stopped at
`Bad Data CRC` and stayed at the U-Boot prompt.

The banner told the story. The bench camera, bought earlier, has a
U-Boot from January 2026. The new one has
`U-Boot 2019.10.0-V4.0.27 (Nov 16 2025 …)`. This build erases **only
the first 4 KiB sector** of the target area, then writes the whole
image. In NOR flash, writing can only turn bits from 1 to 0, so
everything after the first 4 KiB becomes the AND of the old kernel
and the new one. The image is unreadable, and the CRC check catches it.

The camera was not lost: U-Boot, its environment and the
configuration partition are never written by the install. But the
factory kernel was gone, and this U-Boot version would break any image
larger than one sector.

## A relay under 4 KiB

If U-Boot can only write 4 KiB reliably, the first stage has to fit in
4 KiB. The card's `uImage` is now a **standalone relay** (about
80 lines of C, under 4 KiB). U-Boot writes it into the kernel area and
starts it as a "standalone" image: a plain function called by `bootm`,
with U-Boot still fully running. The relay:

1. finds the `fatload` and `bootm` commands **in U-Boot's own command
   table**, in memory around its own return address, so there is no
   fixed address and it works with every U-Boot build;
2. loads the real installer (`tuyabegone-amorce`, a small Linux
   kernel and initramfs) from the card;
3. boots it, or falls back to the U-Boot prompt if anything fails.

The installer then erases and writes the flash **from Linux**, with a
proper driver, whatever U-Boot did or did not erase. It also keeps the
existing behaviour: it waits for a press of the reset button before
writing anything, backs up the flash to the card, takes the sensor
calibration from the factory firmware, writes the configuration and
the firmware.

Validated on the new camera (U-Boot of November 2025) and on the bench
camera (January 2026): about **45 s from the button press** to a camera
running TuyaBeGone.

While waiting for the button, the installer now also **beeps every
10 s** through the camera's speaker, in addition to the heartbeat
LED. With a camera high on a wall, a sound is easier to notice than a
small LED.

## Verified erases over UART

The UART install path (`tbginst`) drives U-Boot directly, and
U-Boot's `sf erase` command has the same limitation. Every erase is now **read back** into RAM and
checked: if the area is not entirely `0xFF`, it is erased again sector
by sector. This is written but **not yet validated** on a camera with
the November 2025 U-Boot, since the SD card path now avoids the
problem.

## A relaxed cross-camera restore

The way back to the factory firmware checks that the image belongs to
this model. For a backup of the camera itself, it still requires an
identical U-Boot. But the two cameras showed that **U-Boot differs
from one production batch to the next**. So when restoring from
another camera's image, only the U-Boot family is required to match,
and the model is still checked through the device tree.

## What the documentation now says

The user documentation lists the known U-Boot versions and what to do
if the relay fails. There is one dangerous case left: an unknown U-Boot
whose command table the relay cannot find. The camera then shows a
solid blue LED and nothing is written to the card. It is unusable until
repaired, because its kernel is already erased. It stays recoverable:
U-Boot and the configuration are untouched, and the serial console (or
a SOIC-8 clip and a programmer) can reinstall everything or restore
the factory firmware. The documentation asks the owners of such a
camera to send a flash image. The first 200 KiB (U-Boot) are enough and
hold nothing personal. The full image contains the factory keys, so it
should only be shared privately.

On the cameras with the November 2025 U-Boot, the restore by SD card
**alone** does not work, because it relies on U-Boot writing large
images. There, use the configuration page or the serial console.

## Status

Version `TBG1.0-261023`: signed package, relay, and the one-file
installer `tbginst.pyz`. Two cameras, two U-Boot versions, every path
tested except the verified UART erase on the newer U-Boot. The source
is still not published.

## About TuyaBeGone

The goal is simple to state: **take an entry-level Wi-Fi cloud camera
and make it local-only**. The LSC cameras sold in Action stores cost
very little and have decent hardware. Out of the box they only work
through a Tuya cloud account and a phone app.

That means the **images, the sound and the camera's data leave your
home** and go to external servers that you do not run. This covers
the live view, the recordings, the motion events, and also when you
are home and when you are not. A camera inside a home sees the most
private part of it. And the app and the service cost nothing. As the
saying goes, **if it's free, you are the product**. At the very least,
once the images are on someone else's servers, you no longer decide
what happens to them, who can see them, or how long they are kept.
The camera also depends on that service: if it changes its terms or
goes away, the camera stops working.

TuyaBeGone removes the cloud instead of trying to block it:

- **no account, no app, no outbound connection**. The camera talks
  only to the local network, and it needs nothing from the Internet
  except the time from an NTP server, which can be your gateway;
- **standard protocols** instead of a vendor app: RTSP, ONVIF profile S
  with motion events, a snapshot URL. It works with an NVR, Frigate,
  Home Assistant or UniFi Protect;
- **the owner stays in control**: setup and installation need a press
  on the camera's physical button, updates are signed, and there is
  always a way back to the factory firmware;
- **no vendor code**: video and audio go through the kernel drivers
  only, and the closed libraries were replaced. The only data reused
  is the camera's own sensor calibration, which is taken from the
  camera itself and never redistributed.

The name says the rest: Tuya, be gone.

{% include coauthors.html %}
