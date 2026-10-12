---
title: "Heating load from a whole-house power meter: measure, model, forecast and colour it in Home Assistant"
date: 2026-10-12 22:00:00 +0200
day: "2026-10-12"
short_title: "Heating load from a power meter, with a forecast"
description: "Using a 5-second whole-house power reading to find a convector's real power, the room's heat loss (about 105 W/K) and its base load, then turning weather forecasts into a heating-load percentage in Home Assistant, with icons coloured by whether the setpoint can be held. Includes a card-mod 4.2 glance gotcha."
tags: [home-assistant, heating, energy, forecast, card-mod, lovelace]
---

**Short version:** with only a whole-house power meter and a thermostat
that switches one plug, we can measure how hard the heating works, model
the room (about **105 W per kelvin** of heat loss, about **620 W** of
non-heating load), and turn the weather forecast into "how much of the
available heating will tomorrow need". On the dashboard, the icon turns
green, yellow, red or purple depending on whether the setpoint can be
held. One of the two convectors is broken, and the numbers show when that
will start to matter.

## What we have

- One room, heated by electric convectors (coefficient of performance 1:
  every watt becomes heat). There are two of them, each controlled by a
  smart plug. One is currently broken: cat hair blocked its airflow and
  its thermal fuse blew.
- A whole-house power reading every 5 seconds. Since all the electricity
  used in the room ends up as heat inside, **total power is a measure of
  the heat produced indoors**, heating or not.
- An indoor temperature sensor, an outdoor one, and a weather
  integration with hourly and daily forecasts.

## What the power meter tells us

**The convector's real power.** Over 67 on/off switches of the plug, the
median step in total power is **980 W** (on: +981 W, off: −979 W). The
power distribution is bimodal: 400–800 W with the heating off,
1400–1800 W with it on. So a 1100 W threshold gives the heating duty
cycle directly from the power reading.

**The second convector never runs.** In a week of data there is no step
of about 900 W outside the first convector's switching, and nothing
around 2400 W. That matched what we knew: it is broken, and its plug is
`unavailable`.

**A daily balance.** Total power is base load + heating. Heat loss is
roughly proportional to the indoor–outdoor difference:

| Day | Total | Heating | Base | Indoor | Outdoor | Heat loss |
|---|---|---|---|---|---|---|
| 09/10 | 1224 W | 612 W | 612 W | 22.6 °C | 9.7 °C | 95 W/K |
| 10/10 | 977 W | 325 W | 652 W | 22.5 °C | 14.1 °C | 115 W/K |
| 11/10 | 1070 W | 442 W | 628 W | 22.5 °C | 12.5 °C | 107 W/K |
| 12/10 | 1058 W | 477 W | 581 W | 22.5 °C | 12.5 °C | 106 W/K |

So about **105 W/K**, which agrees with an older analysis of the same
meter over several years (2.77 kWh per day per °C, i.e. 115 W/K). Base
load is about 620 W.

## The model

The heating power needed to hold the setpoint is:

```
needed = 105 × (setpoint − outdoor) − 620   [W]
load   = needed / available capacity
```

With a 22.5 °C setpoint, the base load alone is enough above about
**16–17 °C** outside. One convector (980 W + 620 W of base load) can hold
at most **outdoor + 15 °C** in steady state:

| Outdoor (multi-day mean) | 1 convector | 2 convectors |
|---|---|---|
| 7 °C | 22.2 °C | 22.5 °C |
| 5 °C | 20.2 °C | 22.5 °C |
| 0 °C | 15.2 °C | 22.5 °C |
| −5 °C | 10.2 °C | 19.6 °C |

For short cold spells, the room's thermal mass helps. We tried to
estimate it from the data and got about 4.7 kWh/K, but with a weak fit
(r = 0.27): with a thermostat, the indoor temperature hardly moves, so
there is little to fit. A simulation of tomorrow's forecast night (down
to 5.4 °C) with one convector gives a drop of only 0.1–0.3 °C for a
thermal mass between 2 and 10 kWh/K. That is why we use the **daily mean**
temperature for "tomorrow", not the coldest hour.

Back-tested on the daily mean outdoor temperatures, the model gives 37 /
13 / 22 % where 31 / 17 / 23 % were measured (with both convectors as the
reference). Right order of magnitude, a few points off. It ignores sun
gains, which makes it pessimistic on sunny days.

## In Home Assistant

All of it is recomputed, nothing is accumulated:

- `sensor.capacite_chauffage_en_ligne`: 980 W × the number of convector
  plugs that are not `unavailable`. When the broken one is repaired and
  back on its plug, every percentage below rescales by itself.
- **Measured load:** (24 h average power − 620) / capacity.
- **Load predicted now:** the model with the current outdoor temperature.
- **Tomorrow / next 7 days:** an automation runs every hour, calls
  `weather.get_forecasts` (hourly and daily), and writes two
  `input_number` helpers. Tomorrow uses the mean of the hourly
  temperatures. The 7-day value is the **most loaded day**, using
  (min + max) / 2 for each day. We had to use an automation here, because
  a template helper created from the UI cannot call a service.
- A `history_stats` ratio helper for the convector's duty cycle over 24 h.

Loads are **not capped at 100 %**. Above 100 %, the setpoint will not be
held, and that is exactly what we want to see.

For the coming month there is no forecast, only climate. Over
2006–2025 (Open-Meteo archive), the period 13 October – 12 November
needs a median of 34 % of both convectors, from 20 % (2022) to 47 %
(2007). With only one convector that is 68 % in a median year, and above
85 % one year in four.

## Coloured icons

On the dashboard, the two forecast values sit in a glance card, and their
icon colour says what will happen:

| Colour | Condition |
|---|---|
| green | load ≤ 80 % |
| yellow | 80–100 % |
| red | the setpoint will not be held |
| purple | setpoint − 1 °C will not be held |

"Setpoint − 1 °C" means a deficit of 105 W, so the purple threshold is
100 + 105 / capacity × 100: 110.7 % with one convector, 105.4 % with two.

### card-mod gotcha: glance entities have their own shadow root

The first attempt put one `card_mod` style on the glance card, with
`.entity:nth-child(3) state-badge { … }`. card-mod rendered the template
correctly, but the icons stayed blue. In the browser, each glance entity
turned out to be a `div.entity` with **its own shadow root** (with card-mod
4.2.1 loaded), so a selector from the card cannot reach the `state-badge`
inside it. The fix is a `card_mod` on each entity entry:

```yaml
- entity: input_number.charge_chauffage_prevue_7_jours
  name: Heating 7 d
  card_mod:
    style: |
      {% raw %}{% set cap = states('sensor.capacite_chauffage_en_ligne') | float(0) %}
      {% set v = states('input_number.charge_chauffage_prevue_7_jours') | float(-1) %}
      state-badge { --state-icon-color:
        {% if v < 0 or cap <= 0 %}var(--disabled-color)
        {% elif v <= 80 %}var(--green-color)
        {% elif v <= 100 %}var(--yellow-color)
        {% elif v <= 100 + 105 / cap * 100 %}var(--red-color)
        {% else %}var(--purple-color){% endif %}; }{% endraw %}
```

We checked the result in a headless Chromium driven over the DevTools
protocol, with the Home Assistant token placed in `localStorage`
(`hassTokens`), then a screenshot.

## What next?

The load is now a number we can forecast. That opens two opposite
questions. We have not tested either of them yet.

### Heating ahead of a cold spell

If the forecast says the convectors will not keep up for a few hours, can
we store heat beforehand? The room's thermal mass is the buffer. With
our rough estimate of 4.7 kWh/K, a 200 W shortfall over a 12-hour night
costs about 0.5 °C. Raising the setpoint by 0.5 °C in the afternoon
before would roughly cancel it. The cost is higher losses while the room
is warmer: about 50 W for 0.5 K, at 105 W/K.

What we need first:

- **A real measure of thermal mass.** Our fit is weak (r = 0.27). A
  deliberate test would do better: a step in the setpoint, or switching
  the heating off for a few hours on a mild night, and watching the
  curve.
- **A rule that is still stateless.** For example: "target = setpoint +
  f(load forecast for the next 12 h)", recomputed every hour, rather than
  a "pre-heating in progress" flag.
- **A comfort limit.** It is a room people live in, not a storage
  heater. A few tenths of a degree is fine, but two degrees is not.

### Lowering the target to save energy and carbon

The other way round: each degree less saves about **105 W** whenever the
heating is running, so about **2.5 kWh per day** and 75 kWh per month of
heating season. The Home Assistant setup already has an Electricity Maps
sensor for the grid's carbon intensity. At tonight's 51 gCO2eq/kWh, one
degree is worth about 4 kg CO2eq per month. That is small, but it does
not cost anything to compute.

Questions we want to answer:

- **When to lower it.** Lowering the target when the load is high saves
  the most energy, but that is also when comfort suffers most. Lowering
  it when nobody is home, or at night, is the classic answer. Can we
  quantify it here?
- **Shifting rather than lowering.** With some thermal mass, the same
  amount of heat can be bought when the grid is cleanest: heat a little
  more when the carbon intensity is low, and coast when it is high.
  This is the same mechanism as pre-heating, driven by a different
  signal.
- **Measuring the result.** The model gives the expected consumption
  for a given outdoor temperature and setpoint. The real meter gives the
  actual one. The difference between the two, accumulated over a
  season, is the only honest measure of what a strategy saved.

Both ideas need finer control than on/off plugs. That is the other
project on the list: variable power on the heating element (triac, whole
mains cycles at zero crossing) and separate control of the convectors'
fans, with an interlock so that the element never runs without airflow.
That is exactly what killed the second convector.

## So, when to repair the second convector?

The practical rule we ended up with: order the replacement thermal fuse
now, and repair when the 7-day icon turns **yellow**. Red only leaves a
few days, which is enough for a repair but not for waiting for a part.

{% include coauthors.html %}
