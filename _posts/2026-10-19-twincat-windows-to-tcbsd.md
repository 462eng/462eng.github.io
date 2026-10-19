---
title: "Moving a TwinCAT 3 machine from Windows to TwinCAT/BSD"
date: 2026-10-19 15:10:00 +0200
day: "2026-10-19"
short_title: "Moving a TwinCAT 3 machine from Windows to TC/BSD"
description: "Field notes from moving a large-format 3D printer (TwinCAT 3 NCI, EtherCAT servos) from the Windows runtime to TwinCAT/BSD on the same industrial PC: engineering version, a corrupted drive startup command, a drive moved to the wrong sync unit, Windows-only calls in the PLC code, and lost persistent positions."
tags: [twincat, tcbsd, beckhoff, plc, ethercat, freebsd, cnc]
---

**Short version:** a large-format 3D printer, driven by TwinCAT 3 (PLC
plus NCI interpolation, EtherCAT servo drives), ran on Windows on a
small Atom-based Beckhoff industrial PC. We moved it to **TwinCAT/BSD**
on the same PC in one day. It now boots into Run and passes its
start-up self-test. These are the traps we hit, without the
machine-specific details.

## 1. The engineering tools must know the target

The engineering environment on the development PC was TwinCAT
4024. It does not know the **TwinCAT OS (x64)** platform, so the
project was built for the wrong target, and activation failed with an
exception in the runtime. We had to move to 4026.

Installing 4026 with the package manager failed halfway: the system
service and the ADS router were missing, so the target could not be
found at all. Installing the base runtime package on its own, then
rebooting, fixed it.

## 2. A newer version is stricter about old project data

After conversion, the 4026 environment crashed when generating init
commands, compiling, or scanning the bus. A minidump pointed at one
**corrupted startup command** in a servo drive's init list, with a
data length of `0xFFFFFFFF`. 4024 had ignored it silently for years.
Removing that one entry from the project file stopped the crashes.

## 3. Check the sync unit of every drive after conversion

The same servo drive then stayed in PREOP (`AL 0x0037`, invalid SYNC1
cycle time). The conversion had attached it to the **1 ms PLC task**,
while its axes run in the **2 ms NC task**. Forcing it back to the NC
sync unit, and putting the two cycle-time parameters back in its
startup list, brought it to OP. Then the whole configuration
activated.

## 4. Windows lurking in the PLC code

A search for Windows dependencies in the code found:

- absolute `C:\` paths for the G-code files. Under TC/BSD the NCI
  folder is `/usr/local/etc/TwinCAT/Mc/Nci`, and any absolute path
  works since the runtime can read any folder. We moved the paths into
  constants;
- `shutdown.exe` started through `NT_StartProcess` on a power failure.
  We replaced it with `NT_Shutdown`, which goes through the TwinCAT
  system service (it still needs a test on the target). The old code
  also **tested the status of a different function block** than the
  one it called, so it never waited for the result, even on Windows;
- a call to lock the Windows session, which has no meaning on a PC
  without a desktop. We removed it.

Everything else (mail, file access, persistent data) worked unchanged.
The operators still drop their G-code into a network share. It is now
a Samba share on the BSD side, limited to private networks in the
firewall. Recent Windows versions block guest access to SMB shares,
which is the first thing to check if an operator PC refuses to
connect.

## 5. Persistent data does not move with the OS

The persistent data (saved encoder positions and offsets for the
vertical axes) stayed behind with the Windows runtime. The machine's
own safety check then refused to start: "axis/encoder offset too
large". That is the right reaction. Re-referencing the axes properly
is a mechanical procedure, not a software one. On this machine the
vertical axes are coupled (gantry) and must never move independently.
For the day, the positions were realigned by hand. We also prepared a
guarded reset step that only rewrites positions, with no motion; it is
not in use yet.

## 6. Slower bus start-up

On TC/BSD the EtherCAT master takes about 24 s to bring the bus up
(probably the long startup list of that servo drive). During that
time the NC is already running, and the linear encoder axes latch an
"invalid I/O data" error (`0x4466`) that stays after the bus is fine.
A reset of those axes once the bus is in OP clears it. Doing it
automatically in the PLC's start-up sequence is still to do.

## Takeaways

- Archive the project before touching anything. Then make a new
  archive at each step, named after what changed.
- Read the target's logs before trying to activate anything.
- Treat a version upgrade of the engineering tools as part of the
  migration, not as a side step.
- After conversion, check sync units and startup lists drive by drive.
- `grep` the PLC code for `C:\`, `.exe` and `NT_StartProcess`.
- Persistent data is the part you lose. Plan the re-referencing with
  someone who knows the mechanics.

{% include coauthors.html %}
