---
title: "Playing on the GPU box from the desk, without a video codec (sRemFB)"
date: 2026-09-28 10:00:00 +0200
day: "2026-09-28"
short_title: "sRemFB: playing on the GPU box from the desk"
description: "sRemFB windowed viewer sremfb-view (SDL3): EVDI virtual display, RAW/LZ4 damage rectangles over TCP, no H.264; keyboard, mouse and Xbox 360 gamepad over uinput; PipeWire audio as PCM over UDP; 8.7 ms key-to-pixel on 2.5 GbE; DROP_MASTER fix."
tags: [sremfb, evdi, latency, pipewire, uinput, gnome]
---

**Short version:** [sRemFB](https://github.com/462eng/sRemFB) now has a
desktop viewer, input forwarding and audio. On a 2.5 GbE link, the delay
from a key press to the pixel arriving at the desk is **8.7 ms median**
at idle. The video is sent as raw or LZ4-compressed damage rectangles,
never encoded to video. Playing up to 60 fps "runs as if it were local"
(the user's verdict).

## Setup

sRemFB is our "network monitor" project. On the server, it creates a
**virtual display** using EVDI, the DisplayLink kernel module. GNOME
treats it as a real monitor that gets plugged in. The server then sends
the damaged rectangles of that display over TCP, raw or LZ4-compressed.
Here the server is the GPU box (Radeon R9700, GNOME 48 on Wayland) and
the client is the workstation at the desk.

The desktop viewer **never uses H.264** or any other video codec. We
chose this because the goal was the lowest possible latency, and a
2.5 GbE LAN has enough bandwidth for most content.

The features below are on a development branch and will be in **the next
release**.

## What's new

### `sremfb-view`, a windowed client

- SDL3, native Wayland.
- VSync off by default. The viewer always presents the **most recent
  frame**; there is no frame queue at any point.
- It plugs a virtual monitor into the remote GNOME session exactly like
  a hardware client does. GNOME remembers its position like any other
  screen.

### Keyboard, mouse and gamepad over uinput

- On the server, input is replayed through **uinput** devices. This is
  **off by default** and enabled with `SREMFB_INPUT=1`.
- Mouse: **absolute** by default (like QEMU's tablet device), or
  **captured relative** mode for games.
- Gamepad: appears as an Xbox 360 pad (`045e:028e`), which Steam and SDL
  recognise.
- On disconnect (or focus loss, or a lost link), **everything is
  released** on the remote side, so no key or button stays pressed.

### Audio

- The server creates a virtual **PipeWire sink**.
- Audio is sent as raw **PCM S16LE, 48 kHz** over UDP, in packets of 128
  frames (2.7 ms).
- The client aims for a **10 ms buffer**. When it gets ahead, it
  **drops samples instead of letting latency drift**.

### Display layout re-applied as *temporary*

mutter's `monitors.xml` is keyed by connector name, and the connector
name changes each time the evdi card is recreated. The server therefore
re-applies the remembered layout itself. It uses the **temporary**
method: a *persistent* `ApplyMonitorsConfig` makes GNOME show "Keep these
display settings?", and GNOME reverts after 20 s if nobody confirms.

### A `DROP_MASTER` fix

sremfb-server keeps a lock fd on each evdi card. When mutter released a
card, this lock fd could **become DRM master** of that card. After that,
mutter's `TakeDevice` failed with `EBUSY`, permanently. The server now
drops master on that fd.

### Living with mutter's GPU hotplug bug

mutter never forgets a removed GPU. That affects evdi cards directly,
and has consequences for VR too; see
[mutter never forgets an unplugged GPU]({% post_url 2026-09-28-mutter-never-forgets-unplugged-gpu %}).
The server creates its cards one at a time. It keeps a card only if
gnome-shell opens it within 1.5 s, and never reuses a number that mutter
refused. In practice, we also leave the server running permanently.

## Measurements

All timings are taken on **the client's clock alone**: input sent, then
the matching pixel **received and decoded** at the client. They do
**not** include the client's own compositor, scanout, or display
latency.

| Case | Result |
|---|---|
| Link (iperf3) | 2.47 Gbit/s on 2.5 GbE |
| Key → pixel received, idle | median **8.7 ms**, p95 **10.9 ms** |
| Remote cursor movement | **0.5 ms** |
| Key → pixel, full-screen noise, RAW (≈210–300 MB/s) | median **~28 ms** |
| Animated desktop | 60 fps at 2.1 MB/s |
| Full-screen noise | ≈36 fps received (link saturated) |
| Audio: click received | median **1.5 ms** |
| Audio: estimated output latency | ~15 ms |
| Audio: long run | 10 min 40 s, no loss, no drift |
| Video latency with audio on | unchanged |

Under full-screen noise, the time is dominated by transmission. One raw
frame is 8.3 MB, which takes about 27 ms at 2.5 Gbit/s. Normal desktop
and game content changes much less of the screen per frame.

In games up to 60 fps, the user reported it "runs as if it were local".

## Not done yet

For very animated content, one idea is **per-tile JPEG**: each tile
encoded as an independent image, with no inter-frame state. This would
cut bandwidth without adding the buffering a video codec needs. It is
**not implemented**.

{% include coauthors.html %}
