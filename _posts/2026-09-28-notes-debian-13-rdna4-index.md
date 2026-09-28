---
title: "Small notes: Debian 13 + RDNA4 + Index"
date: 2026-09-28 10:30:00 +0200
day: "2026-09-28"
short_title: "Small notes: Debian 13 + RDNA4 + Index"
description: "Debian 13 trixie-backports kernel/Mesa/firmware for RDNA4, missing display-manager.service, initramfs udev hook copying .link files recursively, evdi 1.15.1 on kernel 7.1, Proton CompatToolMapping, appmanifest StateFlags 1026, Index camera over UVC/V4L2."
tags: [debian, rdna4, amdgpu, steam, valve-index, notes]
---

Short list of pitfalls from setting up a Debian 13 machine for VR and
gaming. The VR-specific findings have their own posts:
[SteamVR judder and Monado]({% post_url 2026-09-28-steamvr-linux-overlay-judder-rdna4-monado %})
and [mutter's stale GPUs]({% post_url 2026-09-28-mutter-never-forgets-unplugged-gpu %}).

## The machine

| | |
|---|---|
| OS | Debian 13 (trixie) |
| Kernel | 7.1.8 (trixie-backports) |
| GPU | AMD Radeon AI PRO R9700 (Navi 48, RDNA4, gfx1201, DCN 4.0.1) |
| Mesa | 26.1.6 (trixie-backports, RADV GFX1201) |
| Firmware | firmware-amd-graphics 20260810 (trixie-backports) |
| CPU | AMD Ryzen 9 5950X |
| Desktop | GNOME 48 / mutter 48.7, Wayland (exposes `wp_drm_lease_device_v1`) |
| VR | Valve Index, 2 base stations 2.0, Index controllers |
| Runtime | SteamVR 2.17.10, Proton Experimental |
| Virtual displays | evdi 1.15.1 (repackaged, see below) |

## Pitfalls

- **Use trixie-backports for RDNA4.** Take the kernel, Mesa and
  `firmware-amd-graphics` from backports.
- **No `display-manager.service`.** gdm was installed but had never been
  enabled, so `display-manager.service` did not exist.
- **Never keep `.link` backups under `/etc/systemd/network`.** The
  initramfs hook `/usr/share/initramfs-tools/hooks/udev` copies `*.link`
  files from `/etc/systemd/network` **recursively**, and puts them all
  in one directory in the initramfs. Backups in a subfolder therefore
  become active at boot. This happened to us: after installing the new
  kernel, a NIC got the wrong name at boot. Keep backups somewhere else.
- **evdi 1.15.1 is needed for kernel 7.1.** Debian's evdi 1.14.8 does not
  build against 7.1: API errors about `drm_driver.date`, `struct_mutex`
  and `fb_debug_enter`. We repackaged 1.15.1.
- **Proton for all games.** In Steam's `config.vdf`, a
  `CompatToolMapping` entry for app id `"0"` sets the default
  compatibility tool.
- **Installing a Steam game without a screen.** With Steam closed, put an
  `appmanifest_<appid>.acf` with `"StateFlags" "1026"` in the library's
  `steamapps/` directory. At its next start, Steam downloads the
  game.
- **The Index camera is a plain UVC device.** It is a side-by-side
  stereo camera: 1920×960 total, 960×960 per eye. It offers MJPEG or
  YUYV at 15–54 fps. It can be read with V4L2 **while SteamVR is
  running**, as long as SteamVR itself is not streaming from it.
- **`setcap` does not survive updates.** `CAP_SYS_NICE` on SteamVR's
  `vrcompositor-launcher` is lost at every SteamVR update, and the same
  goes for `monado-service` after every `cmake --install`.
- **Index audio goes through DisplayPort.** The sink only exists while
  the headset is on. Use `wpctl set-default` once, and WirePlumber
  remembers it (details in the
  [SteamVR/Monado post]({% post_url 2026-09-28-steamvr-linux-overlay-judder-rdna4-monado %}#index-audio)).

{% include coauthors.html %}
