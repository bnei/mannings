# Normal depth: enumerate the roots, report the shallowest

The solver no longer assumes that discharge increases with depth. It samples `Q(y) − Q` on a
grid over `[0, ceiling]`, bisects every sign change, reports the **shallowest** root, and states
that others exist when they do.

This supersedes the rationale stated until now in `index.html`, `README.md`, `CLAUDE.md` and the
closing paragraph of [0002](./0002-sweep-response-is-the-solve-mode.md): that `Q(y)` is strictly
increasing, so a sign change brackets a unique root. That argument is correct for a trapezoid
and false for the section types this tool is being generalised to carry.

## Why it fails

A circular section's conveyance peaks below the crown, because above that depth the wetted
perimeter grows faster than the flow area. For `D` = 1 m, `n` = 0.03, `S₀` = 0.01:

```
Qmax  = 1.117606 m³/s at y/D = 0.9380
Qfull = 1.038952 m³/s          (92.96% of Qmax)
```

Every `Q` in `(Qfull, Qmax)` therefore has **two** normal depths — roughly a 7% band of
discharge, not a numerical curiosity.

A compound section is worse, because the discontinuity is in the geometry rather than in a
derivative. A 1 m trench opening onto a 20 m bench at elevation 1.0 (`n` = 0.03, `S₀` = 0.01):

```
y = 0.999   Q = 1.6005
y = 1.001   Q = 0.4388      wetted perimeter jumps 19 m, area does not
y = 1.100   Q = 2.6334
```

`Q ≈ 1.5` has three roots. So the real dichotomy is **open versus closed or compound**, not
trapezoid versus everything else, and there is no ordering of section types under which the
monotonicity assumption quietly keeps working.

## Why the shallowest

All the roots are mathematically valid uniform-flow depths. The shallowest is the one the
channel actually reaches when filling from empty under a rising hydrograph, and it is the
conservative answer for freeboard and for velocity. Reporting the deepest, or reporting a set,
would make the primary result something other than a number — and the tool exists to produce
one number a reader can put in a report.

The others are not hidden: when more than one root exists the results panel says so. That is a
statement of fact about the section, not a recommendation, which keeps it on the right side of
[0001](./0001-sensitivity-sweep-not-an-optimizer.md).

## Alternatives rejected

**Bracket on the rising limb only.** Correct for a pipe — find the conveyance maximum, search
below it. Wrong for a compound section, whose first local maximum is the top of the low-flow
trench, far below the real answer for any large discharge. It would give a confidently wrong
number rather than a failure.

**Newton, or a solver per section type.** Newton needs a derivative that a compound section does
not have, and it diverges from a starting point on the wrong side of a peak. A solver per
section type means the number depends on which branch of a switch ran, which is exactly what
becomes indefensible to a reviewer.

## Consequences

- **Grid resolution is a correctness parameter, not a performance knob.** A root pair narrower
  than one grid cell is missed silently. At `GRID` = 200 the circular pair, which spans about 6%
  of the depth range, is found with two orders of magnitude to spare. When arbitrary sections
  land, every breakpoint elevation must be forced into the grid: that is where the
  discontinuities are, and a bench edge can sit anywhere between two evenly spaced samples.
- Cost roughly doubles per solve, and `compute()` runs on every keystroke while the sweep
  re-samples up to 400 points. Measured rather than assumed — see the check-in for step 4.
- The trapezoid yields exactly one bracket, so every existing reference value and the
  round-trip check are unchanged. That is what makes this refactor checkable.
- Bisection is kept, for the same reasons as before: a sign change brackets a root, there is no
  derivative to get wrong, and the iteration cannot diverge. Only the bracketing changed.
