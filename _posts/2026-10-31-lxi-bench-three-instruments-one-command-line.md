---
title: "Our LXI bench: three instruments, one command line, and what they really measure"
date: 2026-10-31 11:00:00 +0100
day: "2026-10-31"
short_title: "The LXI bench and what it can do"
description: "Our test bench is a Rigol DP832A power supply, a DL3031A electronic load and an MSO1074Z oscilloscope, all on the network and driven together by 462lxi, a small Python tool with no VISA library. What the bench can source, absorb and observe, what it does when the instruments are combined, and what a certified multimeter revealed about its readings, including a supply channel that silently sinks 40 mA."
tags: [462lxi, rigol, dp832a, dl3031a, mso1074z, lxi, scpi, vxi-11, test-bench, metrology]
---

**Short version:** three networked Rigol instruments, a power supply,
an electronic load and an oscilloscope, driven as one by `462lxi`, our
command line tool. Together they can source, absorb and watch almost
anything a small electronics project needs. A certified multimeter then
showed that the instrument we trusted most was the one that was wrong,
and that a power supply channel can sink current while switched off.

## The instruments

| Role | Instrument | Ratings |
|---|---|---|
| Power supply | Rigol DP832A | 3 isolated linear channels: 2 × 30 V / 3 A, 1 × 5 V / 3 A |
| Electronic load | Rigol DL3031A | 150 V, 60 A, 350 W; CC, CV, CR, CP |
| Oscilloscope | Rigol MSO1074Z | 4 analog channels, 70 MHz, 1 GSa/s, 8 bits |
| Reference | RISH Multi 18S | certified true-RMS multimeter, used to check the others |

The first three speak LXI: SCPI commands over the network, through
VXI-11. The multimeter is handheld, for now: its infrared link to the
PC is on the to-do list.

## One tool to drive them

`462lxi` is a single Python file with **no dependency**: no PyVISA, no
vendor library. It implements VXI-11 directly over ONC RPC, which takes
fewer than two hundred lines. It reads values the way an electronics
engineer writes them (`3v3`, `4u7`, `2k2`, `500mA`), and every command
can answer in JSON for scripts.

```
462lxi devices                      # are all three online?
462lxi psu set 1 3v3 500mA --on     # voltage and current limit, then on
462lxi load set cc 1A --on          # load in constant current
462lxi scope wave 1 trace.csv       # 1200 points, acquisition keeps running
462lxi --json psu meas              # for scripts
```

Most of its code is not about sending commands, it is about **not
trusting the answers**. Each instrument handles a bad value in its own
silent way: the supply ignores an out-of-range setpoint and replies
`No error`; the oscilloscope clamps it, shows a popup, and loses the
next request; the load, to its credit, refuses with a proper error but
resets its setpoint to zero on every range change, which in CR mode is
a short circuit. So `462lxi` checks every value against the limits it
reads from the instrument, changes load ranges only with the input off
and the setpoint at its minimum, and reads back what it wrote.

## What the bench can do

**Source.** Three channels from 0 to 30 V (5 V on the third), up to
3 A each, with 1 mV / 1 mA setpoint resolution and 0.1 mV / 0.1 mA
readback. Two channels in series give 60 V, or ±30 V around a common
point. Over-voltage and over-current protections per channel. The
supply cannot absorb current: it is a source only.

**Absorb.** Up to 150 V, 60 A and 350 W in constant current, voltage,
resistance or power. The load also has a dynamic mode (current steps up
to 30 kHz, slopes up to 5 A/µs), a 512-step list mode and a battery
discharge mode with capacity, energy and duration counters. Its Sense
terminals measure at the device under test, so the 50 mΩ of bench
cables (50 mV at 1 A) drop out of the measurement.

**Observe.** Four analog channels at 70 MHz, plus logic channels and a
signal generator on this model. `462lxi` reads the displayed trace
(1200 points in about 50 ms, without stopping acquisition) rather than
screenshots, which is 20 to 40 times faster for repeated measurements.

**Combine.** This is where the bench earns its keep. Each important
quantity can be read by two instruments, which checks the whole chain.

| Test | Instruments | How |
|---|---|---|
| Efficiency and regulation of a converter | supply + load | input voltage and load swept point by point, both sides measured |
| Switch-mode supply validation | DUT + load + scope | load in dynamic mode, scope on the output: ripple and transient response |
| Power-on edge | supply + scope | scope armed in single shot, then the channel turned on |
| Current limit check | supply | voltage high, current capped: the channel must switch to CC |
| Real battery cell | supply + load | CC/CV charge, pulsed discharge, capacity and model (see the [battery post]({% post_url 2026-10-31-characterise-an-18650-and-replay-it %})) |
| Simulated battery | supply (+ load) | the supply follows a cell model, measured or generic, new or aged |

A test is a script: it can run unattended for days, write every point
to a text file, and be run again identically.

## What the multimeter found

Two days ago, we
[calibrated the load in software]({% post_url 2026-10-29-rigol-dl3031a-repair-software-calibration %}),
taking the supply as the reference. This week we checked both against
the certified multimeter, in voltage and in current. **The supply was
the one off**:

- channel 2 reads its own voltage **10 to 12 mV high** (within its
  specification of 0.05 % + 10 mV), and its current **1.2 mA + 0.09 %
  high**, even at zero current;
- its output sits 2 to 14 mV **below** the setpoint, and not linearly,
  so correcting the setpoint is not enough: the output has to be
  regulated on a measurement;
- the load, raw, was already right within **±2 mV**. Most of the
  "7 mV + 0.11 %" error we had corrected last time was the supply's.
  Its **+43 mA current offset in CV mode, 15 V range**, on the other
  hand, is real: the multimeter sees it too.

`462lxi` now corrects both instruments from the multimeter readings.
After correction, they agree with it within ±2 mV and ±0.4 mA. This is
not a traceable calibration, but the reference is a certified
instrument, not another bench instrument.

## The channel that sinks 40 mA

The most surprising finding: **a DP832A channel absorbs a constant
37 to 41 mA as soon as the voltage on its terminals is above its
setpoint**, output on or off, whatever the current limit. It does not
show it in its own readback. Connect a charged cell to a channel that
is switched off, and the cell slowly discharges into it. With a weak
source (under 10 mA) it pulls the node down to about 0 V, which looks
exactly like a wiring short.

The only inert state we found: **channel on, setpoint above the node,
current limit at 0 A**. Even then it supplies about 1.2 mA. Until we
have relays to isolate the outputs, the rule on the bench is simple:
unplug a cell from any channel that is not working.

## Two channels, one 60 V supply

The DP832A has no internal series mode: its first two channels are
isolated from each other, and one wire from CH1− to CH2+ puts them in
series. CH2 goes at the bottom, because CH2 and CH3 share their
negative inside the instrument (the manual forbids putting those two in
series or in parallel). The 5 V channel then stays referenced to the
negative of the whole assembly.

`462lxi` drives the pair as a single channel, `1+2`. Each channel gets
half the setpoint and the same current limit, which the manual
requires, and the measured voltages are added. The current is read
from channel 2, the one corrected against the multimeter, since the
same current flows through both.

```
462lxi psu batt 1+2 1Ah 200mA --cells 12 --on
462lxi psu charge 1+2 10Ah 2A --cells 13 --on
```

A guard checks the pair at every reading and stops on any of these:

- one channel switching off alone;
- a channel above its own setpoint, which means it is sinking;
- the two current readings disagreeing, the other sign of a channel
  sinking;
- current limiting, in modes where it is not expected.

On the bench, with the load as a test resistor:

- a simulated 12S cell at **50.34 V**: the two halves within
  **0.2 mV** of each other, and the sum within 2 mV of the load's sense
  and the multimeter;
- the same in battery-simulation mode, with the load absorbing 300 mA
  at the simulated voltage: 13S, 50 V, 15 W, held to **1 mV** for one
  minute;
- load set above the current limit: both channels off within
  **1.5 s**;
- channel 1 switched off by hand during a run: channel 2 off at the
  next reading, **0.25 s** later.

The first 13S run stopped for nothing. A channel sitting exactly on its
setpoint reported itself "unregulated" while sinking nothing: the
current readings from the supply and the load agreed. When current
limiting is expected, the guard now relies on the voltages and the
currents, not on that status.

Two findings came out of the session:

- **Two channels switched off in series both sink.** About 38 mA flows
  through the pair, and the voltage splits unevenly: 36 % on one
  channel, 64 % on the other, whatever the setpoints. Scaled up, a
  48 V pack would put about 31 V on one 30 V channel. So a pack only
  goes on a pair that is already on, at 0 A with its setpoint above
  the pack. Measured in that state, the pair passes **60 µA**, about
  as inert as it gets. Unplug the pack before switching the pair off.
- **Channel 3 cannot see that current.** When CH3 drives current into
  CH2, its own readback shows 0 mA: because of the shared negative,
  the current returns inside the instrument without crossing CH3's
  measurement. Until the multimeter went in series, we believed nothing
  was flowing.

Charging a real pack on the pair is still untested: it passes in
simulation, and waits for a pack.

## Safety rules

The bench drives real hardware, so a few rules apply to every script
and every session, human or not. Know what is wired before turning
anything on. Set voltage **and** current before switching on (at
power-up the supply starts at 0 V but with a 3 A limit). Start low.
With the load, the source comes on first and goes off last. At the end,
outputs off **and** setpoints back to zero, so that an accidental
`on` delivers nothing.

## What next

Expose the supply's 2048-step timer and the load's dynamic and list
modes in `462lxi`; read the multimeter over its infrared link; and add
a relay board, driven over the network, to disconnect outputs instead
of relying on an inert state that is not quite inert.

{% include coauthors.html %}
