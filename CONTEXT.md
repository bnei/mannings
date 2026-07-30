# Manning's — trapezoidal channel

A single-purpose solver for uniform steady flow in a symmetric trapezoidal channel. This
glossary fixes the vocabulary of the domain so the UI labels, the code and the docs cannot
drift apart. It records terms only — no implementation detail.

## Channel geometry

**Bottom width** (`b`):
The flat invert of the trapezoid, in metres. May be zero, which makes the section
triangular.
_Avoid_: base, base width

**Side slope** (`z`):
Horizontal run per unit rise of the channel bank, H:V. `z = 2` means 2 m out for every 1 m
up. May be zero, which makes the section rectangular.
_Avoid_: batter, slope (collides with bed slope), 1:z, V:H

**Channel height** (`H`):
Invert to top of bank — the physical channel, independent of how much water is in it.
_Avoid_: depth, height, bank height, y_max

**Flow depth** (`y`):
Invert to water surface. Distinct from channel height; either may exceed the other.
_Avoid_: depth (unqualified), stage, h

**Top of bank**:
The elevation `H` above the invert. Water above it means the section overtops.
_Avoid_: freeboard line, rim, crest

## Hydraulics

**Normal depth**:
The flow depth at which the channel conveys a given discharge under uniform flow. The
output of the depth mode.
_Avoid_: equilibrium depth, y_n (in prose)

**Capacity**:
The discharge the channel conveys when flow depth equals channel height — flowing full to
top of bank, not overtopping.
_Avoid_: full flow, Q_max, design flow

**Freeboard**:
Channel height less flow depth, when positive. When negative the condition is overtopping
and is named as such, never as negative freeboard.
_Avoid_: negative freeboard

**Near critical**:
Froude number within roughly five percent of unity, where uniform-flow assumptions are
unreliable. Reported as a named condition rather than as exact criticality.
_Avoid_: critical (unqualified), Fr = 1

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
The part of the chart where the channel would overtop: response above channel height in
depth mode, above capacity in flow mode.
_Avoid_: failure region, unsafe zone, overtopping curve
