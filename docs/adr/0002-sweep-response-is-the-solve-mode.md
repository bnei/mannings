# The sweep's response is the solve mode, not a separate control

The sweep chart has no y-axis selector. The response is whatever the current solve mode
produces — normal depth in depth mode, discharge in flow mode — and the only sweep control is
which parameter varies.

The alternative was an explicit response dropdown, which reads as the more obvious design and
makes the feature more discoverable. It was rejected because the mode already determines which
of `Q` and `y` is an input, so a free choice of response makes illegal pairings representable:
depth cannot be plotted against a fixed depth. Inheriting the mode makes those pairings
*unrepresentable* rather than something to validate against and explain, and it means the
fixed-input field on screen is always the right one. It also adds exactly one control to an
inputs panel with no vertical room for two.

The consequence worth knowing is that the legal swept parameters then differ by mode, and each
mode's own fixed input is sweepable:

| Mode | Response | Sweepable |
|---|---|---|
| Solve for depth | Normal depth | `Q`, `b`, `z`, `n`, `S₀` |
| Solve for `Q` | Discharge | `y`, `b`, `z`, `n`, `S₀` |

`H` is absent from both because it appears in neither `Q(y)` nor `normalDepth(Q)` — sweeping it
would draw a flat line. Sweeping the fixed input is the rating curve, which is why it is the
default: the resting state of the chart is the most recognisable figure in the domain, and it
displays the strictly increasing `Q(y)` that the normal-depth bisection's uniqueness argument
depends on.
