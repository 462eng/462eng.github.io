---
title: "TuyaBeGone, part 2: a test bench, a hardware OSD, signed updates and an installer"
date: 2026-10-22 16:00:00 +0200
day: "2026-10-22"
short_title: "TuyaBeGone, part 2: OSD, signed updates, installer"
description: "Follow-up on the free firmware for the LSC (HT-IPC280, Anyka AK3918AV130) camera: an isolated UniFi Protect test bench, a hardware OSD driven through a reconstructed ISP protocol, image flip, camera names in Protect, ECDSA-signed updates from a file or a URL, a way back to the factory firmware, a reusable SD card, and tbginst, a one-file installer working over UART, SD card or both."
tags: [camera, anyka, buildroot, onvif, unifi-protect, embedded-linux, u-boot, firmware-update]
---

**Short version:** a week after the
[first post]({% post_url 2026-10-19-tuyabegone-free-firmware-lsc-camera %}),
the firmware for the LSC camera gained a date and name overlay drawn by
the ISP itself, image flipping, proper names in UniFi Protect, signed
updates and a way back to the factory firmware. A one-file tool now
backs up, installs and restores a camera over the serial console, an
SD card, or both. Everything below was tested on a camera.

## An isolated test bench

Testing a camera against the production NVR means adopting, removing
and re-adopting it many times next to cameras that matter. So we built
a separate bench:

- a spare UniFi Dream Machine Pro, reset and **not linked to any cloud
  account**, with its own Protect;
- its WAN plugged into a spare port of the main firewall, on a new
  interface that only allows Internet access, nothing on the LAN;
- its LAN on a USB network adapter of the workstation, so the bench
  can be watched and scripted from there;
- a Raspberry Pi 3B+ as a minimal 2.4 GHz access point, bridged to
  that LAN (the camera's Wi-Fi is 2.4 GHz only).

Two traps on the Pi with Raspberry Pi OS 13 (trixie). The image ships
NetworkManager's state file with `WirelessEnabled=false`, so the access
point stayed "unavailable" even with rfkill unblocked; `nmcli radio
wifi on` fixes it for good. And the first boot ran before the
`machine-id` was set, so the DHCP client ID changed and the same MAC
got two leases; setting the bridge's client ID to its MAC address
(`ipv4.dhcp-client-id=mac`) gives a stable lease.

## A date and name overlay, drawn by the ISP

UniFi cameras print the date and the camera name in a dark band at the
top left. We wanted the same thing without re-encoding anything, so we
used the **OSD block of the ISP**, which overlays the image before the
hardware encoder sees it.

There is no documentation for it, and the vendor library that drives
it is closed. Static analysis of that library gave us the protocol: one
ISP `ioctl` with sub-commands to enable the overlay and swap its
buffer, a small YUV palette, 2 bits per pixel, and three DMA buffers
allocated through ION so the next frame can be drawn while the ISP
reads the current one. A small C file in our library replays it.

The text is a DejaVu Sans Mono Bold font rasterised at build time into
a C header (Latin-1, four sizes), so the camera needs no font engine.
The band is black at 60 % opacity, the text is white, and its height
follows the stream (about 1/24 of the picture height: 19 px in 360p,
44 px in 1080p). It is redrawn every second on both streams and on the
JPEG snapshot. One detail: the ISP clears all overlay zones each time
it is opened, so the daemon sets them up again after every restart.

Two smaller additions came with it:

- **Image flip** (180°, mirror, vertical flip) for cameras mounted
  upside down. The setting must be applied again after each day/night
  switch, because the ISP's night profile carries its own flip bits.
- A **live view in the web page**, without a plugin: the daemon
  exposes the raw H.264 stream on a local port, and a small JavaScript
  file repackages it into fragmented MP4 for the browser's Media Source
  API. A slow client skips to the next keyframe instead of slowing the
  camera down.

## Names in UniFi Protect

Protect named every camera "manufacturer model", so they all showed up
as the same thing. It reads the model from the WS-Discovery scopes,
then from the ONVIF device information. The camera now:

- announces its configured name in the ONVIF `name` scope;
- reports its Wi-Fi MAC address as the **serial number**, which is
  unique per camera and survives reinstalls. When we reinstalled the
  factory firmware and then TuyaBeGone again on the same camera,
  Protect recognised it without a new adoption;
- answers Protect only (recognised by its HTTP client) with an empty
  manufacturer and the camera name as the model. Other ONVIF clients
  still get the standard answers.

Protect does not read the name again after adoption, so renaming a
camera means renaming it in Protect too. For sound, set the camera's
compatibility mode to *Improved* in Protect and **save** first: the
audio switch only appears after that.

## Signed updates, from a file or a URL

The web page could already take an update file. It can now also fetch
one from a URL: either a small manifest listing the files, or a system
image directly. That works over HTTP or HTTPS, from a local server or
a GitHub release.

Each image comes with a detached **ECDSA P-256 / SHA-256 signature**,
checked on the camera with mbed TLS against a certificate built into
the firmware (a "462eng IoT" publishing key; the private key stays off
the repository). HTTPS is used without certificate checks on the
camera, since the signature already proves where the image comes from.
Downloads are capped at the partition size. A checkbox, ticked by
default, allows unsigned images for development builds; the integrity
checks (squashfs and U-Boot headers, sizes) always run.

Tested on the camera: signed image accepted, unsigned and modified
images refused, truncated image refused, unsigned image accepted only
with the box unticked, HTTPS download from GitHub working.

## Bugs found on the camera

- The upload check measured file sizes with `stat`, which the camera's
  busybox does not have. **Every web update failed** with "incomplete
  upload". It now uses `wc -c`.
- Services restarted from the web page inherited the request's
  environment (all the `HTTP_*` variables exported by the web server).
  They are now started with `setsid env -i`.
- DHCP runs in the background at boot and sometimes finished after the
  ONVIF services started. WS-Discovery and the event server then exited
  ("No usable IPv4 address"), and Protect lost both discovery and
  motion events. They now wait up to 60 s for an address.

## Going back to the factory firmware

The first version could not return to the factory firmware, because
the SD install overwrites the original kernel before it can be saved.
This is now covered from three directions:

- **from the web page**, with a full 8 MB flash backup: the camera
  checks that the backup comes from the same model (same size, same
  U-Boot and device tree as its own flash) and contains the original
  kernel, not ours or the bootstrap. It then copies a static busybox to
  RAM, unmounts the config partition and writes each partition
  directly from the uploaded file, checking an MD5 per partition. The
  kernel is written last, so an interrupted restore still boots
  something;
- **from the installer tool**, over the serial console;
- **from the image of another camera** of the same model, when no
  backup of this one exists. The camera's own config partition is kept,
  since it still holds its original Tuya identity and calibration.

All three paths ended with the Tuya firmware booting again.

## tbginst: one installer, three modes

`tbginst` is a Python tool with a Tkinter GUI and a command line,
shipped as a single `.pyz` file with pyserial inside. The only
requirement is Python 3.8 or later with Tkinter.

| Mode | What it does |
|---|---|
| SD card only | prepares a card; the camera installs from it after a press on the button |
| UART + SD card | prepares the card and follows the bootstrap on the serial console |
| UART only | backs up, installs and configures over the serial console, no card |

Over the serial console it stops U-Boot, sends a **rescue system**
with YMODEM and boots it from RAM. The rescue system is the bootstrap's
kernel and busybox with a shell as init, so it never touches the flash
on its own. It reads the flash in base64 (about 80 s for 8 MB), and the
tool writes partitions back through U-Boot with a CRC32 check before
and after each write. A UART install writes the configuration (Wi-Fi,
access password, name, time zone) directly into the config partition,
so the camera comes up ready.

Serial console traps:

- U-Boot only switches baud rate for `loady`. **1.5 Mbaud works,
  921,600 does not.**
- The kernel's serial driver ignores a switch back to 115,200, the
  console's normal speed. So the rescue system stays at 1.5 Mbaud and
  ends with a reboot.
- The tool followed the bootstrap by matching a French message
  containing "é". In UTF-8 that is two bytes, so the match never
  happened: the install worked but was reported as failed. Matching is
  now accent-independent.

Backups never overwrite each other. Each one goes into its own folder
named after the camera (a hash of its vendor identity file) or the
time. An early version would have written over the folder holding the
backup of the **original** firmware, made by the bootstrap. Worse, the
bootstrap, seeing that folder present, would then have skipped backing
up the original.

## One SD card for many cameras

The bootstrap now writes the firmware itself from the card, kernel
last, without moving or changing anything on the card. The same card
installs one camera after another; the files on the card had the same
MD5 sums after several installs. It also copies an optional settings
file from the card (cameras without a name get a generic one with a
short ID). At the end the LED stays on: remove the card and power
cycle. If the card is left in, the bootstrap sees that the version is
already installed, puts the normal kernel back and boots, without
waiting for the button. An install takes about 35 s after the button
press.

## Status

The bench camera runs the current build, adopted in Protect, with OSD,
flip, motion and sound working. Every installer path (SD only, UART +
SD, UART only, backup, restore from its own backup or from another
camera's image) was tested on that camera. Next is a new camera,
with the serial console connected **before its first power-on**, so
that a full backup is taken before anything else. The source is still
not published.

{% include coauthors.html %}
