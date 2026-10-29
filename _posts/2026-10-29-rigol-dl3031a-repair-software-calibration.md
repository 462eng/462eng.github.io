---
title: "Repairing a Rigol DL3031A stuck in short circuit, and calibrating it in software"
date: 2026-10-29 20:30:00 +0100
day: "2026-10-29"
short_title: "DL3031A: a dead op-amp, then a software calibration"
description: "Our Rigol DL3031A electronic load showed a permanent short on its input. Gate voltages pointed to one MOSFET pair driven fully on; removing its MC33078 driver cleared the short, and the chip turned out dead when tested on its own. While the replacement is on order, our bench tool 462lxi learned to drive the load and to correct its readings, from a 169-point sweep against the power supply."
tags: [rigol, dl3031a, electronic-load, repair, mosfet, op-amp, scpi, 462lxi, calibration]
---

**Short version:** our Rigol DL3031A electronic load had a dead short
on its input, whatever its settings. The power MOSFETs were fine: one
pair was being driven fully on by its op-amp, an MC33078 that turned
out to be dead. The load now runs without that pair until new parts
arrive. Meanwhile, our bench tool `462lxi` learned to drive it, and to
correct its readings in software after a sweep against the power
supply.

## The symptom

The input measured about **40 mΩ, purely resistive**, all the time.
Input on or off, any mode, any setpoint, after a reset, after a
firmware update: no difference. An electronic load that shorts its
source as soon as it is connected is worse than useless, since it can
damage what it is meant to test.

## The power stage

The current is absorbed by IRFP250N MOSFETs working in their linear
region. They are organised in **pairs**, and each pair has its own
**MC33078** dual op-amp, one channel per MOSFET, which sets the gate
voltage so that the current follows the setpoint. The current is read
on a single 0.5 mΩ shunt, followed by a precision amplifier (ADA4522-1).

A short on the input therefore has two possible causes. Either a MOSFET
has failed short, or something is holding a gate high and the MOSFET
is just doing what it is told.

## Following the gates

With the load powered and its input off, every gate should be at zero.
We measured the gate-source voltage of each MOSFET: all at zero, except
**both MOSFETs of one pair, at 8 to 9 V**. That is fully on, and two
IRFP250N fully on in parallel give roughly the resistance we measured.

Both channels of the same chip at the same wrong voltage pointed to the
chip, not to the transistors. We desoldered the MC33078 of that pair,
and **the short disappeared**. With the pair's gates left without a
driver, we ran the usual battery of tests: CC, CV, CR, CP, input on
and off. Everything passed, with current readings within ±5 mA of the
power supply.

## Confirming the culprit

A fault that disappears when a part is removed does not prove that the
part is guilty: something around it could have been driving it wrong.
So we tested the MC33078 on its own, wired as two voltage followers,
powered at ±15 V by two channels of the bench supply. Whatever the
input, **both outputs stayed stuck at the positive rail**. The chip is
dead.

Before fitting a new one, we will check that its surroundings did not
kill it: continuity of the supply pads (pins 4 and 8) to the ±15 V
rails, and the gate resistor and zener of both channels. New MC33078s
and a few spare IRFP250Ns are on order. Until the pair is back, the
load stays well below its rating.

## A load in 462lxi

`462lxi` is the small command
line tool we use to drive the bench instruments over the network (a
Rigol DP832 supply and an MSO1000Z oscilloscope), in plain Python with no
VISA library. It now drives the load as well:

```
462lxi load cc 1.5A           # constant current, 1.5 A
462lxi load on
462lxi load meas -w           # U / I / P / R, refreshed
462lxi load meas --raw        # readings without correction
462lxi load sense on          # measure at the Sense terminals
```

The load has a dangerous habit: **any range change resets the setpoint
of the mode to zero**, and in CR mode a zero resistance is a short. So
`462lxi` only changes mode or range with the input off, and brings the
setpoint to its minimum first. It also reads back the range after
setting it, because `:SOURce:CURRent:RANGe` is silently ignored outside
CC mode. Setpoints are capped at the rated values (150 V, 60 A,
350 W), although the instrument accepts slightly more.

## Measuring the error

Testing the repair showed that the load's readings were off. To know by
how much, `tools/dload_sweep.py` connects the load to channel 2 of the
DP832, with the load's Sense terminals wired to the supply's output, so
that both instruments measure the voltage at the same point. The supply
is the reference.

For each point, the script waits, takes six readings at 0.5 s intervals
on both instruments, and starts again (up to three times) if the first
half differs from the second. It stops at once if the load reports a
protection or turns its input off by itself: a point measured after
that would be wrong and nothing would show it. A few precautions keep
the sequence safe: the source is off between blocks (turning the load
on in front of a charged output capacitor would empty it in one go); in
CV mode the load is turned on first and the source rises with a current
limit; and since the load does not start on a setpoint of 5 mA or less,
it starts at about 20 mA and is then brought down to the point.

169 points later, in all modes and ranges, the picture was clear:

- the **voltage** reads low by about **7 mV + 0.11 %**, in every mode;
- the **current** reads low by about **2 mA** almost everywhere;
- except in **CV mode, 15 V range**, where it reads **43.7 mA too
  high**, constant. It is there even with no current at all: input on
  in CV 1 V, nothing flowing, the load reads +42.7 mA.

We do not know yet whether this offset is older than the failure. The
sweep will be run again once the pair is refitted.

## Why in software

The obvious fix would be to recalibrate the instrument. We read its
calibration data (`:CALibration:DATA:GET?` and friends), but writing it
back is another matter: Rigol's procedure is not public, the command
that opens calibration mode accepts any argument without any way to
check what state it is in, and there is no safe way to rewrite the
tables. Breaking the internal calibration of a working instrument to
fix a 43 mA offset is a bad trade.

So `462lxi` corrects the readings itself. The model is linear,
`read = true × (1 + slope) + offset`, with one set of coefficients for
the voltage, one for the current in CV 15 V, and one for the current
elsewhere. Each set also has a floor: below it, the load reads a clean
zero (input off), and the correction leaves it alone instead of turning
it into minus the offset. After correction, current and voltage fall
within **1 to 2 mA and a few mV** of the supply.

The correction is on by default, and `--raw` gives the instrument's own
values. It assumes that the DP832 is right, which is an assumption and
not a traceable calibration, and that the load does not drift: if
anything changes, the sweep has to be run again and the coefficients
recomputed. The script, the data and the procedure are kept with the
tool, so that takes about twenty minutes.

## What next

Fit the new MC33078 after checking its surroundings, run the full test
battery on all the MOSFETs, then repeat the sweep to see whether the
CV 15 V offset survives the repair.

{% include coauthors.html %}
