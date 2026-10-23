---
title: "HAWK: tomorrow's EDA tool, with yesterday's best idea"
date: 2026-10-23 21:00:00 +0200
day: "2026-10-23"
short_title: "HAWK: tomorrow's EDA, yesterday's best idea"
description: "HAWK, our C++/Qt 6 schematic and PCB editor, brings back the live link between schematic and board that Proteus ISIS/ARES had and EAGLE partly kept, as a single unified document. On top of it, an assisted design mode: Claude works through transactional operations and compact text views, while HAWK places, routes, checks and simulates locally. First test: the review of a real 2020 board."
tags: [pcb, eda, qt, claude, llm, autorouter, erc, ngspice, electronics]
---

**Short version:** HAWK is the schematic and PCB editor we have been
writing since July. It starts from an old idea that most of today's
tools have lost: the schematic and the board are **one design**, not
two files kept in sync by hand. In two days it gained an assisted
design mode. Claude reads compact views of the board and sends
batches of operations. HAWK does all the geometry: placement, routing,
checks and simulation. Its first real job was the review of an
existing board, and it found every design error.

## An idea we had thirty years ago

In the 1990s, Proteus linked its schematic editor (ISIS) and its PCB
editor (ARES) live: a change on one side showed up on the other.
EAGLE kept part of it with forward and back annotation between its
schematic and its board. Most modern tools went the other way: two
documents, and an "update PCB from schematic" step that you run, and
sometimes forget to run.

HAWK (Hardware Assembly & Wiring Kit) goes back to the live link and
takes it one step further: **there is only one document**. Schematic
wires and PCB tracks are both "geometry that implies connectivity". A
connectivity engine compiles them into one shared graph of nets. A
track drawn on the PCB creates the matching connection in the
schematic, and the other way round. There is nothing to synchronise,
so nothing can go out of sync.

HAWK is written in C++ and Qt 6, Linux first, for makers and small
shops. Its native format, `.hawk`, is JSON Lines in millimetres with a
1 nm resolution, written deterministically so that it diffs well in
git. It carries the schematic, the board, the libraries, notes and the
original import sources. HAWK imports EasyEDA Standard and Eagle XML,
and exports back to EasyEDA; our test boards come out identical after
a round trip. It also produces BOM, pick-and-place and Gerber files.

## Claude decides, HAWK computes

The unified model turns out to be exactly what a language model
needs. The goal of the new mode is a **complete board** (schematic,
placement, routing, fabrication files) from a description of the need
and its constraints. The split of work is strict:

| Claude | HAWK |
|---|---|
| Understands the need, picks the architecture and the parts | Local part catalogue, stock, footprint import |
| Writes the schematic **as nets** ("U1.12 → SDA") | Labels, connectivity, electrical rules check (ERC) |
| States placement constraints (groups, edges, sides, proximity) | Placer: legal positions, no overlap, inside the outline |
| Sets the rules (widths per net class, planes, layers) | Autorouter, copper zones, design rules check (DRC) |
| Reads the defects and fixes them | Checks, summaries, simulation |

The model **never draws raw geometry**. Language models are slow,
expensive and bad at placing segments and vertices. They are good at
intent, and since the geometry *is* the connection in HAWK, intent is
enough.

A few rules keep the token count down:

- **Never the raw document.** A real `.hawk` file weighs megabytes,
  around a million tokens. The model sees compact text views (summary,
  component, net, unconnected, constraints…), with a budget. Repeated
  structures are folded: ten identical slots show as one line with
  `×10`, and a plane net is cited once instead of listing every pad.
  A full review of a 300-part board went from 15,500 to 8,400 tokens.
- **Stable names.** References (`U1`), pins (`U1.12` or `U1.SDA`) and
  net names, never internal indices that shift after a deletion.
- **Batches are transactions.** One tool call carries a list of
  operations. If one fails, the board goes back exactly to its previous
  state, and the error names the failing operation. The same
  `applyOps` serves the agent, the command line and the tests.
- **Results are deltas.** Each batch returns what changed and the new
  defects, not the whole state.
- **Append-only conversation.** The system prompt and the tools are
  fixed, the first view of the board is cached, then only deltas are
  added, so the prompt cache works on every turn. Deterministic
  writing helps: the same board always gives the same prefix.
- **A picture when text is not enough.** A `view` tool renders the
  board for the model (millimetre grid in operation coordinates, top
  copper red, bottom blue, ratsnest yellow), about 1,000–1,500 tokens
  per image.

The agent runs inside the editor, in a dock (F3). You can follow its
reasoning and its decisions as they happen, and see each tool it
calls. The board builds up on both views one batch at a time, and
each batch is one undo step. It also runs
from the command line, with a log of every exchange. It talks directly
to the Claude API (Opus 5.5, adaptive thinking); the tests replay
recorded streams offline.

## A home-made router and placer

The **autorouter** is a grid A* (0.1 mm, eight directions) with
PathFinder-style negotiation: only the nets in conflict are ripped up
and retried. It handles any number of layers, inner planes, keep-outs
and differential pairs routed as a ribbon, with serpentines for length
matching.

Its first version routed a real two-layer board of ours (72 nets)
completely, with a clean DRC, in about 5 s, but not well: too many
vias, cut-up ground planes. A human review of that routing was turned
into routing preferences, and the same board came out with 24 signal
vias instead of 52 and an unbroken ground plane. On a 257-footprint
board rerouted in four layers, the router used 159 vias and 4.7 m of
track, against 225 vias and 5.3 m in the human version.

The **placer** is simulated annealing over footprint outlines, with
persistent constraints (group, near, apart, edge, side, fixed, region,
orientation) and a few terms from experience. The final placement is
always legal. On the 72-net board, total wire length (HPWL) was
1,710 mm unconstrained, against 1,741 mm for the human placement, in
3.2 s.

## Electrical profiles

Most PCB tools know that a pin is "input" or "output", and not much
more. HAWK now has **electrical profiles** on nets and pins: kind of
signal, current, voltage, bandwidth, pairs, impedance, and for pins
their state at boot, their safe state and their idle state. From them
HAWK sizes tracks, vias and clearances to IPC-2221, and runs a much
richer ERC, including what happens while the MCU is still in reset.
What is not declared, HAWK tries to guess (part families, pin names,
common MCU families), and every field says whether it was declared,
measured or guessed.

A local **ngspice** run checks a sub-circuit, with boundary conditions
taken from the profiles and generic models. Simple cases (a divider, an
RC corner, an LDO) give the expected values. Switching regulators
without a model are refused rather than guessed.

## The first test: a review

Before letting the agent design a new board, we asked it to **review
an existing one**: a two-board design from 2020, imported from
EasyEDA, with a "review, do not edit" brief. The schematic was a later
revision that already included the fixes made with wires on the
prototype.

The report found **all the real design errors** of the board, with
the right consequences, including some that had survived that
corrected revision. One of them had never caused a visible symptom on
the prototype. Only a check that reads the schematic could find it.

The lessons of each review go into a file the agent reads with its
instructions, and when a lesson can be mechanised, the ERC checks it
directly. The review report itself is kept with the test data: future
versions must keep seeing what this one saw.

## Status

HAWK edits, imports, exports, places, routes, checks and simulates.
The agent can drive all of it, and has reviewed a real board well. It
has **not yet designed a new board end to end**. That is the next
test, on a real new board that will be manufactured. HAWK is not
published yet.

{% include coauthors.html %}
