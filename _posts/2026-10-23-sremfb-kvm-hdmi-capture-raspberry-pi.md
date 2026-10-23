---
title: "sRemFB gets a KVM mode: HDMI capture and USB keyboard on a Raspberry Pi 4"
date: 2026-10-23 20:50:00 +0200
day: "2026-10-23"
short_title: "sRemFB KVM: HDMI in, USB HID out, on a Pi 4"
description: "A new sRemFB backend, sremfb-kvm, turns a Raspberry Pi 4 with a TC358743 HDMI-to-CSI bridge into a KVM over IP for the existing viewer: V4L2 capture, damage computed by tile differences, UYVY conversion, keyboard and mouse sent to the target through a USB HID gadget, and the Pi's hardware H.264 encoder as a fallback. The desktop viewer also learns to decode H.264. Written and tested on a PC against a synthetic source and a fake driver; not yet run on a Pi."
tags: [sremfb, kvm, raspberry-pi, v4l2, tc358743, usb-gadget, hid, h264]
---

**Short version:** [sRemFB](https://github.com/462eng/sRemFB) shows a
remote screen with very low latency by sending raw, LZ4-compressed
changed rectangles. Until now the screen was a virtual display created
on the remote machine, or the display of a virtual machine. With the new **`sremfb-kvm`** backend, it can
be the HDMI output of any machine: a Raspberry Pi 4 captures it, sends
it to the usual viewer, and types on the target as a USB keyboard and
mouse. A BIOS screen or a machine with no network becomes reachable
from the desk. Everything is written and tested on a PC; **nothing has
run on a Pi yet**.

## Where sRemFB comes from

sRemFB ("simple Remote Frame Buffer") started in July with a modest
goal: turn **a small, old single-board computer into an extra
monitor** for a Linux PC. The first board was a Banana Pi M1+, an
Allwinner A20 board from the mid-2010s, connected by Ethernet.
On the PC, the server creates a virtual display with EVDI (the kernel
module behind DisplayLink docks). GNOME treats it like any other
monitor, and the server sends what changes on it to the board, which
writes it to its framebuffer.

Then the project grew, one step at a time:

- **USB teleport** (1.3, July): the keyboard, mouse or any USB device
  plugged into the board becomes a device of the PC, over usbip. The
  small board turned into a remote seat, not just a remote screen.
- **A clean output on the board** (August): a DRM/KMS path with page
  flipping instead of the old framebuffer, validated on the A20.
- **Virtual machines** (1.4, August): a second backend, `sremfb-spice`,
  mirrors the display of a QEMU/KVM virtual machine through SPICE.
  The idea is a fleet of virtual workstations on a hypervisor, each
  desk being only a cheap board with a screen and a keyboard.
- **A desktop viewer, input and sound** (1.5, September): a window on
  another PC instead of a board, with keyboard, mouse and gamepad sent
  back, and the server's audio. That is how we
  [play on the GPU box from the desk]({% post_url 2026-09-28-sremfb-playing-on-the-gpu-box-without-a-codec %}).
- **KVM over IP** (today): the screen of a machine that runs nothing
  of ours at all.

## The fundamentals

Every backend follows the same three rules:

- **It must feel like bare metal.** The person at the desk should not
  notice that the machine is somewhere else: no smearing, no
  compression artefacts on text, no lag on the cursor, games playable.
- **Minimal latency.** Nothing waits for a full frame or a group of
  frames. Changed rectangles leave as soon as they are known, through
  non-blocking queues, and latency is measured all the time (PING/PONG,
  key-to-pixel probe: 8.7 ms median on the desktop viewer).
- **Use all the bandwidth there is.** On a LAN, bandwidth is cheap and
  latency is precious. So the default is raw pixels with fast LZ4
  compression, and the image is degraded only when the link is
  actually congested (RGB565 with dithering, or an H.264 episode),
  then it goes back to raw with a clean snapshot as soon as it can.

## Why a KVM

The EVDI and SPICE backends both need our server running next to the
display, inside a working OS. A KVM over IP needs nothing on the
target: it sees an HDMI monitor and a USB keyboard, from the BIOS
onward. Existing projects such as PiKVM do this well, through a web
browser. We wanted the same viewer, the same protocol and the same
feel as the rest of sRemFB, with no browser.

## The hardware

- a **Raspberry Pi 4 B**;
- a **TC358743** HDMI-to-CSI-2 bridge board on the camera connector;
- the Pi's **USB-C port in device mode**: it is the only USB port of
  the Pi 4 that can act as a gadget, so it carries the keyboard and
  mouse to the target;
- gigabit Ethernet, which the raw path needs at 1080p.

Two hardware details shape the software. On two CSI-2 lanes, the bridge
can do 1080p30 in RGB but **1080p50 in UYVY** (YUV 4:2:2). So we
capture UYVY, and the EDID shown to the target offers 1920×1080 at
50 Hz, plus 1280×720, 1024×768 and 800×600 for firmware setup screens,
never 1080p60. And since the USB-C port is also the Pi's power input,
the Pi has to be powered elsewhere (GPIO header or PoE), with a cable
that does not carry the target's 5 V. The gadget declares itself
self-powered, so that the target does not reserve 500 mA for it.

## The Pi is just another sRemFB server

The Pi runs the **unchanged sRemFB server core**: sessions,
non-blocking queues, RAW/LZ4, RGB565 with dithering, PING/PONG
latency, pressure control, H.264 episodes. The protocol does not
change: blanking, input, H.264 and the hello flags already existed.
Three pieces are new.

**A V4L2 frame source.** EVDI tells us which rectangles changed; an
HDMI capture does not. So the backend compares each frame with the
previous one **by tiles** and builds the damage rectangles itself,
then converts only the changed tiles from UYVY to the wire format.
The bridge does not follow a mode change of the source by itself. On
each source-change event, the backend reads the new timings, applies
them and restarts the stream. Meanwhile the viewer gets a blank screen,
then the new size.

**Keyboard and mouse as a USB HID gadget.** A small configfs script
creates a composite gadget with a keyboard, a relative mouse and an
**absolute pointer**. The viewer already sends Linux key codes for its
other backends. They are mapped to HID usages by a table generated
as the inverse of the kernel's own HID keyboard table (the mapping the
target will apply), and written to `/dev/hidg*`. The absolute
pointer lets the cursor follow the viewer's mouse exactly, without the
drift of relative mice. The relative mouse is used in "captured"
mode (right Ctrl) and for firmware setup screens that do not support
a tablet.

**The Pi's hardware H.264 encoder.** Raw LZ4 stays the default. When
the link is congested, the server can already switch to an "H.264
episode", then come back to raw with a clean snapshot. On the Pi, that
uses the VideoCore encoder through V4L2 memory-to-memory
(`bcm2835-codec`), in High profile at level 4.2. We wrote it from the
driver's source, and a few details are not in any documentation:

- controls have to be set **after** buffer allocation. The driver only
  creates its firmware component on the first `REQBUFS`, and controls
  set earlier are silently ignored;
- the driver rounds the input stride up to 64 bytes (1366 → 1408), so
  frames are copied line by line when the stride differs;
- odd sizes or widths above 1920 are refused, and the client then
  simply stays in raw mode.

## The viewer learns H.264

Until now, the
[desktop viewer]({% post_url 2026-09-28-sremfb-playing-on-the-gpu-box-without-a-codec %})
never used a video codec. It now decodes H.264 with libavcodec
(low-delay mode, slice threads, our own YUV → RGB conversion, BT.601
limited range by protocol convention). It only does so when the server
decides to, under measured congestion. The rule of the project does not
change: raw first, the codec as a fallback. A decoding error, or three
access units in a row without a picture, disables H.264 for the
process, and the viewer reconnects in raw mode. Against a test server
using x264, a 1080p frame costs about 9 ms, 6 of them in the
conversion. A build without libavcodec still works.

## Testing without the hardware

The backend has a **synthetic source** that goes through the same path as V4L2 frames: a test
pattern, a moving rectangle, a frame counter, and on a signal a
simulated mode change to 1024×768 and back, or a target switched off.
It can also replay a raw UYVY capture in a loop. On a PC, the backend
builds with x264 instead of the Pi encoder, so the whole chain (capture,
tile diff, conversion, H.264 episodes, viewer) runs on one desk. The
V4L2 encoder was tested against a fake driver only.

The Pi side is packaged separately, as a layer for our immutable
Raspberry Pi image: a cross-compiled `sremfb-kvm`, the HID gadget, the
EDID, a systemd unit, an example configuration and a diagnostic
script. As with the rest of sRemFB, there is **no encryption or
authentication**, only a trusted LAN. Because a KVM gives full control
of a machine, the address allow-list is **mandatory** in the shipped
configuration.

## Status

The code is merged into the main branch of sRemFB, together with
H.264 in the viewer. Next is the real test on a Pi 4 with the bridge:
capture timings, encoder behaviour, the gadget seen by a real BIOS,
and the latency from a key press to the pixel. Audio over HDMI, virtual
USB storage and power control are left for later.

{% include coauthors.html %}
