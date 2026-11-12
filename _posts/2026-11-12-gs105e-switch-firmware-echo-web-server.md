---
title: "Inside a NETGEAR GS105E: the web server that was only an echo"
date: 2026-11-12 18:05:00 +0100
day: "2026-11-12"
short_title: "A five-port switch, taken apart"
description: "Taking the firmware of a NETGEAR GS105E apart, from the factory HEX file down. No OS, a Dallas 8051 core in flat 24-bit mode, a Broadcom switch. The web interface it seems to carry does not work: the port answers, but it only echoes back whatever you send. And the HTML, images and log strings it stores belong to a different switch, from a different brand — a D-Link, left in the binary and never removed."
tags: [reverse-engineering, firmware, switch, netgear, 8051, ghidra, nsdp, embedded]
---

**Short version:** we pulled apart the firmware of a NETGEAR GS105E, a
small five-port "smart" switch, starting from the factory update file.
There is no operating system: a Dallas 8051 core running flat 24-bit
code, a Broadcom switch chip, a parallel flash. The interesting part is
the web interface. The switch *looks* like it has one — HTML, images,
log messages, a file table — but it does not run: the TCP port answers
and simply **echoes back** whatever you send it. And the web resources
it carries are not NETGEAR's. They are a **D-Link's**, left in the
binary and never taken out.

## No computer in here

A home router runs Linux. This does not. The GS105E is a bare-metal
firmware on a **DS80C390**, Dallas/Maxim's extended 8051, built with
Keil C51 on a uIP network stack. The whole address space is a flat
mebibyte, code and data mixed, no memory banking.

It took a moment to see that. A first pass read the image as a banked
8051 — a common 8051 only addresses 64 KiB, so a 1 MiB image "must" be
paged. It was not: the regular 32 KiB boundaries that looked like bank
windows were an artefact. The giveaway is in the first three
instructions after reset, which write the Dallas *timed-access*
sequence (`AA`, `55`) and then set the address-control register to
contiguous 24-bit mode. From there every long jump and every data
pointer is four bytes with a 24-bit operand, and the correct
disassembly only falls out under Ghidra's `80390` flat-mode language.
Read the same bytes as 16-bit and you get plausible-looking nonsense.

The automatic analysis only reached about 40 % of the code — the rest
is only called indirectly — so we closed it by hand: make a function at
every call target, then iterate to a fixed point. That turned 741
functions into 4 956 in one pass.

## What the hardware is, read from the strings

You can read most of the bill of materials off the firmware without
opening the case:

- the flash announces itself, through the bootloader, as an **EON
  EN29LV800CB**;
- the switch core is a **Broadcom RoboSwitch** (`go BCM SW SPI`,
  `enable broadcom header`, register-access timeouts), reached over
  both MDIO and SPI;
- the PHY is a **BCM5464**, named in a cable-diagnostics routine that
  measures fault distance per pair.

There is also a **factory test console** on the serial port, with a
small command set — read and write the MAC address and serial number,
read the model and firmware version — which is how each unit gets its
identity programmed. The identity is not in the factory image: it is
written per unit, so an official update never overwrites it.

## The web server that is not one

This is the part that sent us down the rabbit hole.

The firmware contains everything an embedded web interface needs: the
uIP stack, log strings like `web server running...`, CGI handler names,
a table of file names (`index.htm`, `top.htm`, `form.css`, `logo.gif`),
and the files themselves — two JPEGs and some HTML, sitting in flash.
It looks complete.

It does not run. Measured on a real unit:

- TCP port 80 is open and accepts the connection;
- but the switch **returns, byte for byte, whatever you send it** —
  7 bytes for 7, 256 binary bytes unchanged, 64 nulls returned as
  64 nulls. A real web server handed `GET / HTTP/1.0` replies with
  `<html>`, not with `GET / HTTP/1.0`;
- send 2000 bytes and exactly 1300 come back. 1300 bytes is the size of
  one uIP buffer: the application returns what fits in a single segment
  and drops the rest.

Two conclusions fall out of that. Nulls and raw binary come back
intact, so there is **no HTTP parser** left. And the 1300-byte
truncation means the application keeps **no state** between segments.
It is the textbook one-line uIP echo application:

```c
if (uip_newdata()) uip_send(uip_appdata, uip_len);
```

The binary backs this up: not one HTTP response string anywhere on the
whole mebibyte — no `HTTP/1.`, no `200 OK`, no `Content-Type` — and no
code that references the stored file names or their offsets. The pretty
`index.htm` is dead data.

## Whose web site is it anyway?

The two JPEGs settle where all this came from. One is a 1024×75 banner
carrying a **D-Link** logo and its old tagline; the other is a small
photo of a **D-Link DES-1526**, a 24-port smart switch. The HTML title
string still in the flash reads *24-Port 10/100Mbps + 2 Combo
Copper/SFP PoE Management Switch* — the DES-1526 exactly.

So the base firmware of this little five-port NETGEAR is a **D-Link
DES-1526** firmware (or the shared ODM base it came from). The HTML, the
images, the file table and the `web server running` messages are a
*different switch, from a competitor*, embedded whole and never
stripped — right alongside unused code for PoE and for fibre ports this
five-port copper box does not have.

It is not even a degraded feature: the first-generation GS105E was
never meant to be managed through a browser. It is configured over
NETGEAR's own discovery protocol, NSDP, with the ProSAFE Plus utility.
The browser interface was never part of *this* product; it is leftover
scaffolding from somebody else's.

The firmware wears its haste in other small ways: `botton block`,
`Agnet MAC address`, `inset tlv struct`, a `cgi_dumy` handler, files
called `.gif` that are really JPEGs, a bootloader that says "18
sectors" for a flash that has 19. None of it matters to a switch that
forwards packets perfectly well — which it does.

## The management protocol

What the switch *does* speak is NSDP, over UDP, called SCCD inside the
code. The firmware handles the full set of configuration messages —
VLANs, port statistics, QoS, rate limiting, mirroring, cable test — and
carries in plain text the well-known shared authentication key of the
ProSAFE Plus line, which lets a client configure the switch without the
admin password. The factory login is the equally well-known
`Default` / `password`.

Reading that protocol off the wire was the one genuinely reusable
outcome, so it lives on its own now, as a small command-line tool:
[github.com/462eng/nsdptool](https://github.com/462eng/nsdptool).

## Where we stopped

Everything that can be done **without opening the case** is done: the
architecture, the flash map, a complete disassembly, the management
protocol, and the proof that the web interface is an echo with a
competitor's clothes on. The firmware also hides an update back door
reachable from the management protocol, and a bootloader with a TFTP
client that would be the way in for custom code.

We are not going further. Taking it past this point means soldering
onto the serial port and flashing the thing, with a real risk of a
one-way trip, to modify a cheap old switch that already does its job.
It stays as it is — a thoroughly documented little box — and the write-up
stays an archive piece.

{% include coauthors.html %}
