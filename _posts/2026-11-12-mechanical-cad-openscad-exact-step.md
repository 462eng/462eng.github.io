---
title: "A HAWK for mechanical CAD: OpenSCAD in, exact STEP out"
date: 2026-11-12 18:00:00 +0100
day: "2026-11-12"
short_title: "A HAWK for mechanical CAD"
description: "The same idea as HAWK, moved to mechanical design: the source is OpenSCAD, our own interpreter evaluates it, and a geometry kernel turns it into an exact B-rep, so cylinders and spheres come out as real STEP surfaces, not facets. Claude writes clean parametric code and decides the design; the tool computes the geometry, the assemblies, the kinematics and the drawings. The first real test is a humanoid robot arm of about fifty parts."
tags: [cad, openscad, step, occt, mechanical, claude, llm, geometry, kinematics, assembly]
---

**Short version:** after [HAWK]({% post_url 2026-10-23-hawk-pcb-design-claude-decides-hawk-computes %})
for electronics, we did the same thing for mechanical CAD. The source
of a part is **OpenSCAD code**; our own interpreter evaluates it and a
geometry kernel builds an **exact B-rep**, so a cylinder exports as a
true cylindrical face in STEP, not a faceted one. The same split as
HAWK: Claude writes the code and decides the part, the tool computes
and shows the result. It does assemblies, moving mechanisms, collision
checks and 2D drawings. Its first real test is a humanoid robot arm of
about fifty printed and bought parts.

## Why not just OpenSCAD

OpenSCAD is a wonderful way to describe a part: it is code, so it
diffs, it is parametric, and a language model writes it well. But its
geometry is **mesh-only**. A cylinder is a prism with as many facets as
you asked for; export it to STEP for machining and you get a prism, not
a cylinder. For parts that will be 3D-printed that rarely matters; for
anything going to a mill, a lathe or a laser, it does.

So we kept the language and replaced the engine. Two pieces:

- **Our own OpenSCAD interpreter**, not a binding to OpenSCAD. It is
  checked against the real OpenSCAD as an oracle: on the language
  tests, the BOSL2 library and a few dozen real parts, **122 of 122
  renders match**. We only depend on OpenSCAD to prove we agree with
  it.
- **A geometry kernel on OpenCascade (OCCT)** that keeps the model as
  an exact boundary representation whenever it can. Cylinders, spheres,
  fillets, extrusions and revolutions stay exact, and the STEP carries
  the real surfaces. When an operation cannot stay exact — a boolean on
  two very faceted shapes, a `hull`, a `minkowski` — it falls back to a
  mesh kernel (Manifold), and the part is **clearly marked `APPROX`**
  rather than pretending. 2D is Clipper2, text is HarfBuzz. On the
  geometry oracle, 43 of 43 real parts match.

The rule we settled on, the only real divergence from OpenSCAD, is
that an explicit `$fn` of 8 or less means *you want a real polygon* (a
hex nut pocket, a square boss), while a default resolution means *you
want a smooth curve*, which we then keep exact. The best of both: a
`cylinder()` is a true circle in the STEP, a `cylinder($fn=6)` is a
genuine hexagon.

## Claude decides, the tool computes

Same division of labour as HAWK, and for the same reason: a language
model is good at **intent** and bad at placing vertices.

| Claude | The tool |
|---|---|
| Understands the part, picks the architecture, the process, the fits | Evaluates the code, builds exact or mesh geometry |
| Writes clean parametric OpenSCAD: named dimensions, a Customizer tab per concern | Reports: bounding box, volume, validity, per-line primitive boxes |
| Chooses clearances and tolerances from the process (printed, machined, laser) | Applies the trade rules for that process, flags what breaks them |
| Exposes the right parameters and checks the extremes | Renders views, measures exact distances and diameters |
| Reads the report and fixes the real defect | Never asks the model to measure a picture |

The model never eyeballs a dimension off an image. Views are for
judging the look; every exact number — a hole position, a remaining
wall thickness, an interference — comes from the geometry, not from
counting pixels. When the user clicks a face in the 3D view, the tool
hands the model the exact feature *and the source line that produced
it*, so a request like "chamfer this edge" lands on the right line of
code.

## Assemblies

A part is one file; an **assembly is a folder**. A placement file
positions the parts, a shared dimensions file holds every interface
dimension written exactly once (a bolt circle, a clearance, a plate
thickness), the parts live one per file in their own fabrication frame,
and bought parts — screws, bearings, motors, profiles — are not drawn
but pulled from a library, with their interface dimensions taken from
the datasheet rather than from memory. The tool keeps a **bill of
materials** in step with the assembly, checks for **interferences**,
and exports the whole thing, including a true assembly STEP.

A common library ships the usual hardware — ISO M2 to M12 screws, nuts
and washers, ball and linear bearings, aluminium profiles, NEMA 17 and
23 motors, pins and magnets — with dimensions from the standards, so a
part's clearances are computed from real interface figures, not copied
numbers.

## Mechanisms that move

The newest part is **kinematics**. A moving part is placed from a
*joint*: a slider in the code, with a range and a comment that says
what it is and in what unit. The kinematic chain is written in the
code too — a reducer makes one joint drive another, a carried part
follows the one that holds it — so the rest pose is just the source
values, and any pose is reached by setting the joints.

The user drives those joints in the 3D view: buttons, or Ctrl-drag a
part and the tool works out which joint moves that point while
disturbing the fewest others, then follows the mouse along its path.
Claude drives them too, through two tools: one sets a combined pose and
checks for collisions against the rest position, the other sweeps a
single joint through its whole range and reports where it collides, so
a declared travel can be proven free before anyone trusts it. Only the
pairs of parts that actually move relative to each other are checked, a
nominal contact (a shaft in its bore) is not a collision, and poses are
rendered on a fast path that moves the cached meshes rather than
rebuilding the geometry.

## Drawings

For anything going to a machinist, the tool produces a **2D drawing**
from the exact geometry: first-angle projection, visible edges solid
and hidden edges dashed (hidden-line removal on the OCCT model, not on
a mesh), an isometric view, overall and intermediate dimensions, hole
diameters grouped (`4× Ø 5.4`), fillet radii, a title block with
material, process, scale and the git revision. It is honest about its
limits: it lists, under "not dimensioned", every feature it recognised
but could not dimension automatically — an inclined face's angle, a
countersink, a groove — so the drawing is never passed off as a
complete definition when it is not.

## The first real test: a robot arm

HAWK's first job was to review a real board. This tool's first real job
is to **design and carry a real assembly**: a humanoid robot arm, about
fifty parts, from the shoulder to the fingers.

[![A rendered humanoid robot arm, posed with the elbow bent and the
hand half open: a shoulder block carrying two stepper motors and a
cycloidal reducer, a white printed forearm shell, a wrist, and an
articulated hand with fingers]({{ '/assets/2026-11-12/robot-arm.png' | relative_url }})]({{ '/assets/2026-11-12/robot-arm.png' | relative_url }})

*The test assembly, rendered by the tool itself and posed through the
joint tools: shoulder flexed and abducted, elbow bent, fingers
half-closed. The gold blocks are NEMA 17 motors, the pale shells are
printed PETG, the hand runs on small serial-bus servos. The whole chain
— shoulder, elbow, wrist, hand — moves from joints declared in the
code.*

The arm exercises everything at once: printed structural parts with
their fits, bought motors, bearings and servos from the library, a
cycloidal reducer whose faceted discs fall back to the mesh kernel and
are marked `APPROX`, a kinematic chain of ten joints with their travel
limits, and collision checks that find the real mechanical stops — the
elbow bottoms out against the upper arm at a true angle, and the tool
reports it rather than letting the geometry pass through itself.

## Status

The interpreter, the geometry kernel, the assemblies, the library, the
kinematics and the drawings all work, and Claude can drive all of it
from a Claude Code session. The name is deliberately still a
placeholder, so we are not using it here. The tool is not published
yet. As with HAWK, the next milestone is a part designed from a bare
need and then actually made.

{% include coauthors.html %}
