# Manning's — channel sections

A single-purpose solver for uniform steady flow in a prismatic channel. This glossary fixes
the vocabulary of the domain so the UI labels, the code and the docs cannot drift apart. It
records terms only — no implementation detail.

## Section types

**Section type**:
The family a channel's cross-section belongs to — trapezoidal, circular, arbitrary. Selects
which geometry parameters exist and whether the section is open or closed.
_Avoid_: shape, geometry, profile, conduit type, cross-section type

**Cross-section**:
The shape of a particular channel, drawn to scale. A section type with its parameters filled
in.
_Avoid_: section (unqualified), profile

**Section properties**:
Flow area, wetted perimeter, top width, hydraulic radius and hydraulic depth evaluated at one
flow depth. A property of the cross-section *and* a depth, never of the cross-section alone.
_Avoid_: section (unqualified), hydraulics, geometry

**Open section**:
A section type whose flow area grows without bound as depth increases — trapezoidal, and
arbitrary sections up to the top of their data.
_Avoid_: unconfined, free-surface section

**Closed section**:
A section type with a crown, above which no further flow area exists — circular.
_Avoid_: conduit, pipe (a circular section may be a culvert or a tunnel), confined

## Channel geometry

**Bottom width** (`b`):
Trapezoidal only. The flat invert of the trapezoid, in metres. May be zero, which makes the
section triangular.
_Avoid_: base, base width

**Side slope** (`z`):
Trapezoidal only. Horizontal run per unit rise of the channel bank, H:V. `z = 2` means 2 m out
for every 1 m up. May be zero, which makes the section rectangular.
_Avoid_: batter, slope (collides with bed slope), 1:z, V:H

**Diameter** (`D`):
Circular only. The internal diameter, in metres. Also the invert-to-crown distance, so a
circular section has no separate channel height.
_Avoid_: bore, size, width

**Channel height** (`H`):
Open sections only. Invert to top of bank — the physical channel, independent of how much
water is in it. A closed section has a crown instead, fixed by its geometry.
_Avoid_: depth, height, bank height, y_max

**Flow depth** (`y`):
Invert to water surface. Distinct from channel height; either may exceed the other.
_Avoid_: depth (unqualified), stage, h

**Top of bank**:
Open sections only. The elevation `H` above the invert. Water above it means the section
overtops.
_Avoid_: freeboard line, rim, crest

**Crown**:
Closed sections only. The highest point of the section's boundary, above which there is no
further flow area. Water reaching it means the section surcharges.
_Avoid_: soffit, obvert, top, roof

**Containing dimension**:
Whichever of channel height or crown elevation bounds the section — the depth at which an open
section overtops or a closed section surcharges.
_Avoid_: max depth, full depth, capacity depth

## Hydraulics

**Normal depth**:
The flow depth at which the channel conveys a given discharge under uniform flow. The output
of the depth mode. A section whose conveyance does not increase monotonically with depth may
have several; the shallowest is reported and the existence of the others is stated.
_Avoid_: equilibrium depth, y_n (in prose)

**Capacity**:
The discharge the channel conveys at its containing dimension — flowing full to top of bank or
to crown, not overtopping or surcharging. For a closed section this is less than the maximum
discharge the section can convey.
_Avoid_: full flow, Q_max, design flow

**Maximum discharge**:
Closed sections only. The greatest discharge the section conveys at any depth, occurring below
the crown. Exceeds capacity, and is reported as a caveat on it rather than as the capacity.
_Avoid_: peak flow, true capacity, Qmax (in prose)

**Freeboard**:
Open sections only. Channel height less flow depth, when positive. When negative the condition
is overtopping and is named as such, never as negative freeboard.
_Avoid_: negative freeboard

**Depth to crown**:
Closed sections only. Crown elevation less flow depth — clearance to the point of surcharge.
Not freeboard, which is clearance against a bank that water can pass over.
_Avoid_: freeboard, headroom, air space, ullage

**Overtopping**:
Open sections only. Flow depth above the top of bank. The section still conveys flow, with the
banks treated as continuing upward, and results are reported with that stated.
_Avoid_: flooding, failure, surcharge

**Surcharge**:
Closed sections only. Flow depth reaching the crown. Flow becomes pressurized and is no longer
governed by a free surface, so it lies outside this tool's scope and is reported as a
condition rather than as a result.
_Avoid_: overtopping, full flow, pressurized flow (as a synonym — it is the cause, not the
condition)

**Near critical**:
Froude number within roughly five percent of unity, where uniform-flow assumptions are
unreliable. Reported as a named condition rather than as exact criticality.
_Avoid_: critical (unqualified), Fr = 1

**Nearly closed**:
Closed sections only. Flow depth above nine tenths of the crown, where the free surface is
narrow enough that hydraulic depth diverges and the Froude number and flow regime cease to be
meaningful. Reported as a named condition, as near critical is.
_Avoid_: almost full, near full, unstable flow

## Sensitivity sweep

**Sensitivity sweep**:
Holding every input fixed but one, evaluating the solver across a range of that one input,
and presenting the result as a curve. It reports a response, not a recommendation.
_Avoid_: optimization, optimisation, solver (collides with the point solver), sensitivity
analysis (implies statistical variance propagation, which this is not)

**Swept parameter**:
The single input that varies across the sweep — the x-axis.
_Avoid_: independent variable, free variable, varying parameter

**Response**:
The computed quantity plotted against the swept parameter — the y-axis. Determined by the
solve mode, never chosen separately: depth mode responds with normal depth, flow mode with
discharge.
_Avoid_: output, dependent variable, objective (implies optimization)

**Rating curve**:
The sweep of discharge against flow depth — the stage–discharge relationship for the
section. Produced by sweeping the fixed input in either mode, so it appears as depth
against `Q` in depth mode and as `Q` against depth in flow mode. Same relationship, axes
transposed.
_Avoid_: stage-discharge curve, Q-y curve, calibration curve

**Operating point**:
The single sweep sample corresponding to the values currently in the input fields — the one
point on the curve the results panel describes.
_Avoid_: current point, design point (implies a design has been chosen), solution

**Exceedance region**:
The part of the chart where the channel would overtop or surcharge: response above the
containing dimension in depth mode, above capacity in flow mode.
_Avoid_: failure region, unsafe zone, overtopping curve
