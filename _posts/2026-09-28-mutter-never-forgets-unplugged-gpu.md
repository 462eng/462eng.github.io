---
title: "mutter never forgets an unplugged GPU (and what it does to VR)"
date: 2026-09-28 09:30:00 +0200
day: "2026-09-28"
short_title: "mutter never forgets an unplugged GPU"
description: "mutter never removes hot-unplugged GPUs: stale wp_drm_lease_device_v1 globals, Xwayland keeps the drm_fd, DRM minors leak up to card63, evdi cards, SteamVR and Monado DRM lease hangs; untested mutter RFC patch and tested Monado patch (XRT_COMPOSITOR_WAYLAND_DRM_NODE)."
tags: [mutter, gnome, wayland, drm-lease, evdi, vr, monado]
---

**Short version:** mutter's native backend adds GPUs on hotplug but never
removes them. Each secondary GPU that goes away (here: DisplayLink/evdi
virtual cards) leaves a stale `wp_drm_lease_device_v1` global behind.
Xwayland keeps the DRM fd it received from that global, so the kernel
never frees the DRM minor. Eventually VR runtimes that enumerate lease
devices hang, or lease the wrong device. Upstream issue:
[GNOME/mutter#710](https://gitlab.gnome.org/GNOME/mutter/-/work_items/710),
open since 2019. Below are an **untested** mutter patch (RFC) and a
**tested** Monado patch that works around the problem.

Context: GNOME 48 / mutter 48.7 on Wayland, kernel 7.1.8, an AMD RDNA4
GPU driving a Valve Index, plus evdi 1.15.1 virtual displays created by
[sRemFB]({% post_url 2026-09-28-sremfb-playing-on-the-gpu-box-without-a-codec %}).

## Root cause (verified in the source)

We read mutter 48.7 and current `main` (50/51); the code paths below are
the same in both.

- `src/backends/native/meta-backend-native.c` connects only to udev
  `device-added` (in `init_gpus`). There is no
  `meta_backend_remove_gpu()`: once a GPU is added, it stays in the
  backend's list.
- In `meta-kms.c`, `on_udev_device_removed()` only calls
  `handle_hotplug_event (kms, NULL, META_KMS_RESOURCE_CHANGE_NONE, __func__)`,
  which re-probes the existing devices. So `kms->devices` and the GPU
  list never get shorter.
- `on_udev_device_added()` deduplicates devices by path. When the kernel
  gives a new device a `/dev/dri/cardN` number that was already used,
  mutter rejects it:
  `Failed to hotplug secondary gpu '%s': device already present`.
- `meta-drm-lease.c` `update_devices()` compares its list with
  `meta_kms_get_devices()`. Since that list never loses the removed
  device, the `wp_drm_lease_device_v1` global is **never withdrawn**.
- **Xwayland keeps the `drm_fd`** it received from each lease device. We
  counted 28 `/dev/dri/cardN (deleted)` fds in Xwayland. The kernel only
  frees a DRM minor when its last reference is dropped
  (`drm_minor_alloc_release`), so every evdi add/remove cycle uses up a
  new `cardN` number, up to `card63`.

We did not find an upstream merge request for this.

## What it does to VR

What we saw on our machine:

- **35 to 37 stale `wp_drm_lease_device_v1` globals.** The count was
  exactly card0 plus every evdi card ever created in the session.
- **SteamVR** got stuck right after `LeaseDevice_Done` and restarted in a
  loop.
- **Monado hung forever.** It waits for the `done` event of every lease
  device, and stale devices never send it.
- Some stale globals **advertised the real GPU's connector** (DP-1,
  "VLV Index HMD", same connector id). Monado sometimes picked one of
  them and failed with `Failed to lease connector`.
- Rebooting, or starting a new session, clears all of it.

evdi causes a separate problem for **SteamVR**, even without stale
devices: when evdi cards are present, SteamVR's lease breaks with
`dc_stream_state is NULL` followed by `flip_done timed out`, at the
moment gnome-shell opens the evdi cards. **Monado** does not have this
problem. We tested starting an sRemFB display while in VR, and restarting
Monado three times with evdi present, at 120 Hz: no errors. (For the
SteamVR side, see
[SteamVR on Linux: overlay/UI judder on RDNA4]({% post_url 2026-09-28-steamvr-linux-overlay-judder-rdna4-monado %}).)

## Patch 1: mutter, forget hot-unplugged GPUs (RFC, untested)

<div class="notice" markdown="1">
**Status: request for comments. This patch has never been run.** It
compiles and was built into a Debian package (`48.7-0+deb13u1+sremfb1`),
but **we have not installed or tested it**. Possible problems:
use-after-free in paths we did not review, and removal of the
**primary** GPU is not handled (the patch logs a warning and does
nothing). Do not deploy it on a machine you depend on.
</div>

- Download: [`mutter-48.7-native-remove-hot-unplugged-gpus.patch`]({{ '/assets/patches/mutter-48.7-native-remove-hot-unplugged-gpus.patch' | relative_url }})
  (against mutter 48.7, 12 files, about 290 added lines)
- Licence: **GPL-2.0-or-later**, same as mutter (it is not covered by
  this site's CC BY-SA licence).

What it does:

- adds `meta_backend_remove_gpu()` and an `on_udev_device_removed()`
  handler in the native backend, for non-primary GPUs only;
- adds `meta_kms_remove_device()` and a new `MetaKms::device-removed`
  signal;
- adds `meta_kms_impl_device_mark_removed()`, which closes the device and
  **prevents it from ever reopening its path**, since the kernel may give
  that path to a different device;
- connects the DRM lease manager to `device-removed`, so the global is
  withdrawn. Xwayland and other clients then close their fd, and the
  kernel can free the minor;
- releases the renderer's per-GPU data once no onscreen uses it anymore.

## Patch 2: Monado, ignore stale lease devices (tested)

- Download: [`monado-ignore-stale-lease-devices.patch`]({{ '/assets/patches/monado-ignore-stale-lease-devices.patch' | relative_url }})
  (against Monado `main` 9950e2a5f, file
  `src/xrt/compositor/main/comp_window_direct_wayland.c`)
- Licence: **BSL-1.0**, same as Monado.

Changes:

- wait at most **2 s** for all lease devices to send `done`, then
  **ignore the ones that did not** (with a warning), instead of blocking
  forever;
- if a connector was requested (`XRT_COMPOSITOR_WAYLAND_CONNECTOR`) and
  is not found, **fail** instead of falling back to the first connector.
  With evdi displays present, the fallback would lease one of them;
- new option **`XRT_COMPOSITOR_WAYLAND_DRM_NODE`** to use only one DRM
  node, because stale devices can advertise the real GPU's connectors.

Tested on our machine in the conditions described above (stale globals
present, evdi cards present): Monado starts and leases the Index at
120 Hz (see the numbers in the
[SteamVR/Monado post]({% post_url 2026-09-28-steamvr-linux-overlay-judder-rdna4-monado %})).

## Workarounds that don't need a patched mutter

In [sRemFB]({% post_url 2026-09-28-sremfb-playing-on-the-gpu-box-without-a-codec %}),
the server now creates evdi cards one at a time. It keeps a card only if
gnome-shell opens it within 1.5 s. Numbers that mutter refuses are kept
as "dead" cards and never used, so the next card gets a fresh number.

The simplest workaround: **never remove evdi cards during a session.** We
now leave `sremfb-server` running all the time. No cards are removed, so
no stale lease devices are created.

{% include coauthors.html %}
