---
title: "SteamVR on Linux: overlay/UI judder on RDNA4, and why Monado fixes it"
date: 2026-09-28 09:00:00 +0200
day: "2026-09-28"
short_title: "SteamVR overlay judder on RDNA4, and Monado"
description: "SteamVR without async reprojection on Linux; enableLinuxVulkanAsync and 120 Hz end in amdgpu flip_done timed out and a watchdog abort; Monado with steamvr_lh, xrizer and WayVR runs at 90/120/144 Hz; XRT_COMPOSITOR_DESIRED_MODE, XRT_NO_STDIN, lighthouse base stations, Index audio over DisplayPort."
tags: [vr, steamvr, monado, amdgpu, rdna4, linux]
---

**Short version:** on our Debian 13 / RDNA4 / Valve Index setup, SteamVR
runs without asynchronous reprojection, and every attempt to enable it
(or to run at 120 Hz) ends in a `flip_done timed out` and a killed
compositor. Switching the runtime to Monado, still using SteamVR's own
Lighthouse driver, gave smooth rendering at 90, 120 and 144 Hz.

For the hardware and software versions, see the
[Debian 13 + RDNA4 + Index notes]({% post_url 2026-09-28-notes-debian-13-rdna4-index %}).
In short: kernel 7.1.8, Mesa 26.1.6 (RADV), AMD Radeon AI PRO R9700
(Navi 48, gfx1201), GNOME 48 on Wayland, SteamVR 2.17.10, Proton
Experimental.

## The symptom

When you turn your head, the world renders smoothly. But anything drawn
as a UI layer judders: VRChat menus, name tags, the HUD, and desktop
overlays (WayVR). The same hardware on Windows does not show it.

## No async reprojection on Linux

Every time `vrcompositor` starts, its log contains:

```
Async support disabled by user setting or a direct mode driver
```

So by default SteamVR on Linux runs without asynchronous reprojection.
There is a setting for it, `steamvr.enableLinuxVulkanAsync`. It is listed
in `resources/settings/default.vrsettings` and defaults to `false`.

We had already applied the usual prerequisite,
`setcap CAP_SYS_NICE+ep` on `vrcompositor-launcher`. Note that this
capability is lost at every SteamVR update and has to be set again.

With the setting enabled, the log shows `Enabling async support!`, and
then:

```
WaitForPendingPresent: failed to wait for present
```

On the kernel side:

```
amdgpu ... [CRTC:428:crtc-1] flip_done timed out
```

Then SteamVR's watchdog kills the compositor:

```
Failed Watchdog timeout in thread Render in RenderDistort after 5.9 seconds. Aborting.
```

Requesting **120 Hz** (without async) gives the same signature. We saw
it on kernels 6.17.8 and 7.1.8. When a lease was granted, we sometimes
also saw `dc_stream_state is NULL for crtc '1'`. At **90 Hz without
async**, SteamVR is stable, but the overlays still judder.

On one occasion we also saw a GPU page fault in the gfxhub (SQC data,
`PERMISSION_FAULTS`), raised from `vrcompositor`'s RenderThread and
followed by a gfx ring reset. We have not found the cause. A RADV/RDNA4
issue is a possibility, but this is **only a hypothesis**.

## Monado + SteamVR's Lighthouse driver + xrizer

The alternative was [Monado](https://monado.freedesktop.org/) (built
from `main` at commit 9950e2a5f, 2026-09-27), with:

- the **`steamvr_lh`** driver (`STEAMVR_LH_ENABLE=true`). It loads
  SteamVR's own Lighthouse driver, so tracking works the same way as
  in SteamVR;
- **[xrizer](https://github.com/Supreeeme/xrizer)** v0.5, which
  translates OpenVR to OpenXR for games like VRChat. The
  [LVRA wiki](https://lvra.gitlab.io/) recommends it, since
  OpenComposite has not been maintained since 2024;
- **WayVR** v26.8.0 for desktop overlays (`wayvr --openxr`).

For Proton games, add this launch option so the container can see the
host OpenXR runtime:

```
PRESSURE_VESSEL_IMPORT_OPENXR_1_RUNTIMES=1 %command%
```

### Measured results

- `hello_xr`, 60 s runs at **90, 120 and 144 Hz**: no `flip_done` and no
  `dc_stream_state` errors in the kernel log. The only missed frames
  were 2 at startup.
- VRChat: "extremely smooth, no judder" (the user's own report after
  playing), with working audio.

### Choosing the refresh rate

Monado lists the Index's Vulkan display modes in this order (the order
may differ on other setups, so check Monado's log):

| index | mode |
|---|---|
| 0 | 90 Hz |
| 1 | 144 Hz |
| 2 | 120 Hz |
| 3 | 80 Hz |

To pick one, set `XRT_COMPOSITOR_DESIRED_MODE=<index>`. Without it,
Monado uses 144 Hz. To see the mode the kernel actually set:

```sh
sudo grep -A12 '^crtc\[.*crtc-1' /sys/kernel/debug/dri/0/state
```

(the CRTC name depends on your system).

### What we conclude, and what we don't

The same GPU, kernel and headset can run 120 and 144 Hz reliably under
Monado. So the `flip_done` failure is most likely **triggered by
SteamVR's presentation path**, and is not a general limit of amdgpu
with the Index. This is a strong hypothesis, not a root cause: we did
not find what SteamVR does differently.

## Monado pitfalls we hit

- **Running under systemd.** `monado-service` fails at startup with
  `epoll_ctl(stdin) failed` / `XRT_ERROR_IPC_MAINLOOP_FAILED_TO_INIT`.
  Setting `XRT_NO_STDIN=1` fixes it.
- **Capability.** `cap_sys_nice=eip` on `monado-service` has to be set
  again after every `cmake --install`.
- **Base stations found only at startup.** Monado looks for Lighthouse
  devices for about 5 s after it starts, and does not add base stations
  it sees later. After power-up, the base stations need about 60 s
  before they can be found. If Monado starts too early, it finds none.
  Our workaround: restart the service if no
  `Found lighthouse base station` line appears in the log.
- **`hello_xr` exits as soon as stdin closes.** For unattended runs:
  `sleep 60 | hello_xr -g Vulkan2`.
- **Stale DRM lease devices** left behind by mutter can block Monado
  forever. That has its own post:
  [mutter never forgets an unplugged GPU]({% post_url 2026-09-28-mutter-never-forgets-unplugged-gpu %}).

## Index audio

The Index's audio goes through DisplayPort, i.e. the GPU's HDA function
("Navi 48 HDMI/DP"). The PipeWire sink
(`alsa_output.pci-…hdmi-stereo`) only exists while the headset is
powered on; its ELD reports "Index HMD". WirePlumber kept using another
output, so we ran `wpctl set-default <sink id>` once. WirePlumber stores
the preference by node name and switches back to it automatically each
time the headset comes back.

{% include coauthors.html %}
