---
title: "Characterising an 18650 cell, then replaying it on a power supply"
date: 2026-10-31 11:10:00 +0100
day: "2026-10-31"
short_title: "An 18650 measured, modelled and replayed"
description: "Our LXI bench can now charge a real lithium-ion cell to within a millivolt, measure it with pulsed discharges, fit a model with two polarisation branches that replays the whole discharge to 3.5 mV RMS, and make the power supply behave like that cell, new or aged. First results on four old Samsung INR18650-25R cells, two worth keeping and two worn."
tags: [462lxi, battery, li-ion, 18650, battery-model, ocv, esr, polarisation, battery-simulation, rigol]
---

**Short version:** with only a power supply and an electronic load, our
bench now charges a real Li-ion cell, measures it, turns the
measurements into a model, and makes the power supply behave like that
cell, as new or aged. On an old Samsung INR18650-25R, the model replays
the real discharge within **3.5 mV RMS**. The bench itself is described
in [the companion post]({% post_url 2026-10-31-lxi-bench-three-instruments-one-command-line %}).

## Why bother

A device that runs on a battery is usually tested on a bench supply,
which is a perfect battery: it never sags, never runs out, never ages.
The real cell does all three, and the interesting bugs live there:
brown-outs under a current peak, a low-battery threshold that triggers
too late, a device that will not start on a cell at 20 %. We wanted to
reproduce those on the bench, on demand, with a real cell's behaviour
rather than a textbook curve.

## Charging to the millivolt

`462lxi psu charge` charges a real cell in CC/CV from one supply
channel. The trick is where the voltage is measured: **the load reads
it**, input off, its Sense terminals on the cell. The supply's setpoint
is then raised (up to +200 mV) to make up for the cable drop, so that
the cell itself, not the supply terminals, reaches the end-of-charge
voltage. Precharge at C/10 below the minimum voltage, constant current,
then constant voltage until the current drops to C/20. Li-ion, LiFePO4,
LTO and lead-acid are accepted; NiMH and NiCd are refused, since they
need a −ΔV or temperature termination.

The first full charge of our 25R took 1868 mAh (7.50 Wh) at 2.5 A. The
surface temperature, read by hand with the multimeter's PT100 probe,
peaked at 32 °C at the end of the constant-current phase and then
decreased in CV. A later storage charge to 3.60 V at 3 A never went
more than **0.8 mV** above its target, while the supply setpoint rose
to 3.689 V to compensate for the cables. A commercial charger is
typically specified to ±50 mV.

At the end of a charge, the channel **stays on at the end voltage with
a 0 A limit**. Switching it off would be worse: an idle DP832A channel
sinks about 40 mA from whatever is above its setpoint (see the
[bench post]({% post_url 2026-10-31-lxi-bench-three-instruments-one-command-line %})).

`--storage` charges to the chemistry's storage voltage (3.60 V for
Li-ion, roughly 30 to 40 %). It is now how every Li-ion cell on the
bench is put away.

## Measuring the cell

`tools/batt_characterize.py` discharges the cell through the load in
**1 A pulses**, each followed by a rest. Steps are at most 5 % of the
capacity, shorter in the knees of the curve. At the end of each rest,
the relaxed voltage gives one point of the open-circuit voltage curve,
OCV(SoC). The jump when the current stops gives the series resistance,
ESR(SoC). The charge counted down to the cut-off voltage gives the
capacity at the test current.

On our cell (32 steps, 5 min rests, 4 h 50 min):

- **capacity 2077 mAh** at 1 A down to 2.5 V, about 83 % of the
  2500 mAh rating (which is specified at 0.5 A, so the real state of
  health is a little higher);
- **OCV** 4.16 V full, then 140 to 270 mV below our generic Li-ion
  curve between 90 and 20 %: the cell is old, and the generic curve is
  only a rough average;
- **ESR** 80 to 134 mΩ. This includes the contacts of the cell holder,
  since the sense wires are on its terminals, so it cannot be compared
  to the ≤ 18 mΩ (AC, 1 kHz) of the datasheet.

The script checks itself first: `--virtual` runs the whole procedure on
a simulated cell. On the eight built-in chemistries, it finds the OCV
within 2 mV on average and the ESR within ±10 %.

## The part a simple model misses

A cell model of the form *OCV(SoC) − ESR × I* matches the voltage at the
start of each pulse within a few millivolts. But under load, the voltage
keeps sinking for minutes, as lithium diffusion lags behind the
current, and recovers just as slowly at rest. That simple model ended
each 1 A pulse about 40 mV too high, and up to 300 mV too high below
5 %.

So the characterisation now fits two **R-C polarisation branches** on
the whole recorded series, pulses and rests, with time constants found
on a grid (here 5 s and 60 s) and resistances that depend on the state
of charge. The branches are kept only if they at least halve the error,
otherwise the fit tends to invent them to absorb interpolation noise.

[![Measured discharge of the 25R in 1 A pulses with 5 minute rests,
from 4.16 V down to 2.5 V over almost five hours, with the model drawn
on top. Below, the model error: about 50 mV at the end of each pulse,
over 150 mV near empty, without the R-C branches; a few millivolts with
them]({{ '/assets/2026-10-31/25r-replay.png' | relative_url }})]({{ '/assets/2026-10-31/25r-replay.png' | relative_url }})

*The recorded discharge replayed through the fitted model. Grey: the
same model with its two R-C branches removed. Orange: the full model,
3.5 mV RMS over the whole run, with a mean error of 2.5 mV or less in
every state-of-charge band. The remaining spikes are the instant the
current switches.*

## Replaying the cell

The model is a small JSON file. `462lxi psu batt --model` then makes a
supply channel follow it: the output tracks OCV − ESR × I − polarisation,
with the state of charge counted from the measured current, and cuts
off at the minimum voltage under load, like a protection circuit would.

```
462lxi psu batt 2 1A --model 25r.json --on            # the cell as measured
462lxi psu batt 2 1A --model 25r.json --soh 80 --on   # the same cell, aged
462lxi psu batt 2 7Ah 3A --chem pb --cells 12 --on    # 2 × 12 V lead-acid
```

`--soh 80` ages the cell: capacity × 0.8 and resistances doubled (the
law is 1 + 5 × (1 − SoH), deliberately simple until we can fit it on
worn cells).

A supply cannot absorb current, so on its own it can only simulate a
discharge. With `--sink`, the load joins in: the supply pushes a
constant current, and the load, in CV at the simulated voltage, absorbs
whatever the device under test does not take. The battery current is
the difference of the two, in both directions, so a charger can be
tested against a simulated cell. Checked with another supply channel as
a 100 mA charger: the simulation counted the charge within 0.13 mA of
the real current, and held its voltage within ±1.4 mV of the target.

We also closed the loop the other way: the load characterised a cell
simulated by the supply (20 mAh, 200 mΩ) and found its ESR within −5 %
and its OCV within 2.9 mV on average. The 3 % error on capacity in that
test is what led us to the supply's 1.2 mA readback offset, now
corrected.

## Four cells, sorted

The first cell was one of four 25R salvaged together. The other three
went through the same chain over the following day: charge,
characterisation, storage charge to 3.60 V.

| Cell | Pulses / rest | Capacity to 2.5 V | Step resistance at 50 % | Replay error (RMS) | without R-C |
|---|---|---|---|---|---|
| a | 1 A / 5 min | 2079 mAh (83 %) | 95 mΩ | 3.5 mV | 18.8 mV |
| b | 1 A / 10 min | 2129 mAh (85 %) | 89 mΩ | 4.0 mV | 17.4 mV |
| c | 2 A / 3 min | 1615 mAh (65 %) | 131 mΩ | 9.7 mV | 37.5 mV |
| d | 2 A / 3 min | 1490 mAh (60 %) | 139 mΩ | 10.2 mV | 41.4 mV |

*Capacity relative to the 2500 mAh rating. Step resistance: the voltage
jump when the pulse stops, divided by the current, holder contacts
included.*

Cell b took a 3 A charge without complaint: 4.20 V reached in 22
minutes, 79 minutes in total, ending at C/20 (124 mA). Its slower test (7.5
hours, 10 minute rests) gave the best curve of the four: with longer
rests, the voltage gets closer to true equilibrium before each OCV
point is read. Cells c and d were run faster, 2 A pulses and 3 minute
rests, about 2 h 15 min each. That costs some accuracy, the replay
error rises to about 10 mV, and a little capacity, since a cell gives
less at 2 A than at 1 A. It does not change the verdict: c and d have
lost a third of their capacity and their resistance is up by half.

The faster test also changes how the fit splits the resistance. With
2 A pulses, the fit finds a 2 s branch, and that branch takes part of
the step resistance: the "instant" series resistance it keeps is only
17 to 83 mΩ, against 77 to 146 mΩ for a and b. The step resistance in
the table is measured the same way on all four, so that is the column
to compare.

So a and b are a matched pair, close in capacity and resistance, good
for a 2P group; c and d are worn, and still a pair for a low-drain
use. Each now has its own model file, so the device a pack is meant
for can be tried on the bench against the real cells it will get.

## Chemistries and pack organisation

Eight generic chemistries are built in, as rough typical curves from the
literature, to be replaced by measured cells. All values are per cell.
With its two 30 V channels in series, the supply reaches 60 V at 3 A,
which sets the number of cells in series. How the two channels are
paired, and what the pair does when it is switched off, is in the
[bench post]({% post_url 2026-10-31-lxi-bench-three-instruments-one-command-line %}#two-channels-one-60-v-supply).
Simulation on the pair has been checked on the bench up to 50 V. Charging
a real pack on it has not been tested yet.

| Chemistry (`--chem`) | Full → empty (open circuit) | Cut-off under load | Charge on the bench | Max in series, simulated | Max in series, charged |
|---|---|---|---|---|---|
| Li-ion NMC/LCO (`liion`) | 4.20 → 3.00 V | 3.0 V | CC/CV to 4.20 V, storage at 3.60 V | 14S | 14S |
| LiFePO4 (`lfp`) | 3.45 → 2.50 V | 2.5 V | CC/CV to 3.65 V | 17S | 16S |
| Lithium titanate (`lto`) | 2.70 → 1.80 V | 1.8 V | CC/CV to 2.80 V | 22S | 21S |
| Lead-acid (`pb`) | 2.12 → 1.97 V | 1.75 V | CC/CV to 2.40 V (absorption) | 28S | 24S (4 × 12 V) |
| NiMH (`nimh`) | 1.40 → 1.05 V | 1.0 V | no (needs −ΔV or temperature end) | 42S | — |
| NiCd (`nicd`) | 1.36 → 1.05 V | 1.0 V | no (same) | 44S (43S with `--sink`) | — |
| Alkaline (`alk`) | 1.60 → 0.95 V | 0.9 V | primary cell | 37S | — |
| Zinc-carbon (`znc`) | 1.60 → 0.90 V | 0.9 V | primary cell | 37S | — |

That covers the usual 48 V packs: 13S or 14S Li-ion, 16S LiFePO4,
four 12 V lead-acid batteries.

For the alkaline and zinc-carbon cells, the resistance rises steeply as
they empty (0.15 to 0.6 Ω and 0.35 to 1.5 Ω for an AA), so the
capacity they deliver depends heavily on the current, and ageing them
with `--soh` makes no sense.

**Series** is native: `--cells N` multiplies the voltages, in
simulation (`psu batt`) as in a real charge (`psu charge`).

**Parallel** is not a separate option, but it is only a matter of
scale: P cells in parallel hold P times the charge, with every
resistance divided by P. Give the pack capacity on the command line,
and `--esr-factor` scales the series resistance and both polarisation
branches together, keeping their variation with the state of charge.
A 2P3S pack of our measured 25R cells:

```
462lxi psu batt 2 4158mAh 3A --model 25r.json --cells 3 --esr-factor 0.5 --on
```

The factor replaces the ageing law, so an aged pack takes the product
of both: the same pack at 80 % health is `--soh 80 --esr-factor 1`
(resistances × 2 for ageing, ÷ 2 for the parallel group; the capacity
is multiplied by 0.8). Whatever P, the simulation stays limited to 3 A per
channel.

**Real packs.** The characterisation treats a pack as one big cell:
give it the cut-off voltage of the whole pack, and the load handles up
to 150 V, 60 A and 350 W. Charging cells in series on the bench assumes
matched cells or a pack with its own protection circuit: the bench sees
only the total voltage, and does not balance.

## Limits

- There is no temperature measurement yet: a cell under test must be
  watched.
- The simulated ESR follows the measured current with 0.5 to 1 s of
  delay, so short current peaks are not reproduced; the slower
  polarisation is.
- With `--sink`, the load dissipates the full bias current at all
  times, and the device cannot draw more than that current.
- The ageing law is a placeholder until worn cells are measured.

## What next

Sense blades clamped inside the cell holder, so that the holder's
contacts drop out of the ESR; temperature logging with an automatic
thermal stop; a full cycle tool (charge, rest, discharge, efficiency);
and, once the relay board is in, unattended multi-day cycling.

{% include coauthors.html %}
