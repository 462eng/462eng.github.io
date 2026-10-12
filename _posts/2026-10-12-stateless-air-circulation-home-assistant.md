---
title: "A stateless air-circulation loop in Home Assistant (and why the AC stayed off)"
date: 2026-10-12 21:00:00 +0200
day: "2026-10-12"
short_title: "A stateless air-circulation loop in Home Assistant"
description: "Why a smart plug stayed off after a power cut (power-on state), and how we drive a fan from CO2 and VOC with Home Assistant helpers recomputed every minute: one helper per pollutant with its own hysteresis, an OR helper, and an actuator that follows it."
tags: [home-assistant, co2, voc, sen66, zigbee, automation]
---

**Short version:** the air conditioner did not switch off because of an
automation. It was off because its smart plug was set to come back
**off** after a power cut. We then replaced the "who turned it on?"
logic we first wrote with a loop that has no hidden memory: every
minute, Home Assistant recomputes a few boolean helpers from the sensors,
and the plug follows them.

## Why the AC was off

In the afternoon, the logbook has an 11-minute gap with no "Home
Assistant stopped" event before it, then a restart. The plug came back as
`unavailable`, then `off`, with no context: no automation and no user
did it. The plug has a `power_on_state` select, set to `Off`. After a
loss of mains, it stays off. A clean restart of Home Assistant the day
before had left it on, because the plug itself never lost power.

The fix is one setting: `power_on_state` → `LastState`.

## What the AC is used for

In this room, the air conditioner mostly works as a fan. Its own
setpoint is above the heating setpoint, so in autumn it just moves air.
The air-quality sensor is a Sensirion SEN66 (CO2, VOC index, NOx index,
particulates).

Over the last 7 days, CO2 had a median of 496 ppm and a 99th percentile
of 926 ppm. The VOC index had a median of 179 and a 99th percentile of
473. Note that the SEN66 VOC index is **relative**: it adapts to the
recent baseline over hours, so a pollution that lasts slowly drifts back
towards 100. It is good at catching a rise, not at measuring a level.

## First version: a flag, rejected

The first automation was event-driven. It turned the plug on when CO2 or
VOC crossed a threshold, set a flag saying "the automation did this",
and only turned the plug off later if the flag was set and the air was
clean. It works, but the flag is state that can drift away from reality:
a power cut, a restart, a manual switch, and the flag no longer matches
the plug.

The verdict was short: "I hate stateful."

## Second version: helpers recomputed every minute

The current version is one automation that runs every minute and at
startup. It does not react to events. It recomputes everything from the
current sensor values:

| Helper | Turns on | Turns off |
|---|---|---|
| `air_circulation_co2` | CO2 > 900 ppm | CO2 < 700 ppm |
| `air_circulation_cov` | VOC index > 350 | VOC index < 150 |
| `air_circulation` | co2 **OR** cov | — |

Then the plug follows `air_circulation` **OR** the existing manual
"cooling" boolean, so that cooling requested by hand is never switched
off. A command is only sent when the plug's state differs from the
target, so the Zigbee network does not get a command every minute.

The only memory left is in the two per-pollutant helpers: between the
two thresholds, each one keeps its previous value. That is what
hysteresis is, and it cannot be avoided. But it is visible in the UI,
and it cannot disagree with anything: if it is wrong, the next pass
corrects it from the sensors.

The core of the automation (YAML view of what was created through the
API):

```yaml
triggers:
  - trigger: time_pattern
    minutes: "/1"
  - trigger: homeassistant
    event: start
actions:
  - variables:
      co2: "{% raw %}{{ states('sensor.aqm_co2') | float(0) }}{% endraw %}"
      cov: "{% raw %}{{ states('sensor.aqm_voc_index') | float(0) }}{% endraw %}"
  - action: >-
      input_boolean.turn_{% raw %}{{ 'on' if co2 > 900 or (is_state('input_boolean.air_circulation_co2', 'on') and co2 > 700) else 'off' }}{% endraw %}
    target: { entity_id: input_boolean.air_circulation_co2 }
  - action: >-
      input_boolean.turn_{% raw %}{{ 'on' if cov > 350 or (is_state('input_boolean.air_circulation_cov', 'on') and cov > 150) else 'off' }}{% endraw %}
    target: { entity_id: input_boolean.air_circulation_cov }
  - action: >-
      input_boolean.turn_{% raw %}{{ 'on' if is_state('input_boolean.air_circulation_co2', 'on') or is_state('input_boolean.air_circulation_cov', 'on') else 'off' }}{% endraw %}
    target: { entity_id: input_boolean.air_circulation }
  - variables:
      wanted: "{% raw %}{{ 'on' if is_state('input_boolean.air_circulation', 'on') or is_state('input_boolean.loft_cool', 'on') else 'off' }}{% endraw %}"
  - if:
      - condition: template
        value_template: "{% raw %}{{ states('switch.climatiseur') in ['on', 'off'] and states('switch.climatiseur') != wanted }}{% endraw %}"
    then:
      - action: "switch.turn_{% raw %}{{ wanted }}{% endraw %}"
        target: { entity_id: switch.climatiseur }
mode: single
```

## Why a helper and not the device

`air_circulation` says "the air needs moving". It does not say which
device does it. Today it is the air conditioner. Later it will be a
dedicated fan that also opens a ventilation damper, and only the last
step of the automation will change. A helper is also an obvious place to
add an on/off switch for the whole function later, read by the
automation but never written by it.

A side effect worth knowing: switching the plug by hand no longer
sticks. Within a minute, the loop puts it back where the helpers say it
should be.

{% include coauthors.html %}
