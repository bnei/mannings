# Section properties sum over disjoint wetted regions

Every section type that ships has exactly one wetted region at every depth, and three places
depend on that without saying so:

- `math.outline(v, d)` is contracted to return **one** polyline, ordered left rim → invert →
  right rim and cut off at elevation `d`.
- `drawSection` closes the water polygon by drawing the surface line from `wet[0]` to
  `wet[wet.length - 1]` — the first and last points of that single polyline.
- `sectionProps` derives `Dh = A / T` from a single top width.

A station/elevation section breaks all three the moment its data has two low points separated by
a higher one — a natural channel with a mid-channel bar, a divided ditch, a road cross-section
with a gutter each side. Take `pts = (0,1), (2,0), (4,0.6), (6,0), (8,1)`. Below a water surface
of 0.6 there are **two** wetted regions; at 0.6 they merge into one. Neither state is exotic and
the section is a plausible thing to survey.

Left alone, the failure is silent and wrong in both halves of the page: `drawSection` runs a
water surface straight across the dry bar, and a naive left-to-right walk of the boundary counts
the bar as wetted, inflating `A` and `P`.

The decision:

- **`props` sums** `A` and `P` over regions, and `T` is the sum of the surface widths. `R = A/P`
  and `Dh = A/T` are then computed from those sums in `sectionProps`, which does not change.
- **`outline` returns an array of polylines.** For the trapezoid and the circle that is an array
  of one, and their inner shape is unchanged.
- **`drawSection` draws one water polygon and one surface line per region.**
- **Froude number and flow regime are suppressed when the region count exceeds one**, reported as
  a named condition alongside near critical and nearly closed.

## Why sum rather than subdivide

One Manning application over the summed properties is a simplification, and the point of choosing
it is to be able to name it as one. The alternative is conveyance subdivision — computing
`K = (1/n)·A·R^(2/3)` per subsection and summing `K` — which is the standard treatment of a
compound section and what a hydraulic model does.

It is rejected for the same reason compound sections are out of scope generally: it needs a
subdivision rule. Where the vertical division lines go, and whether they count toward wetted
perimeter, are conventions, and they differ between references. A number that depends on an
unstated convention is not defensible to a reviewer without a footnote explaining which one ran,
and the tool exists to produce one number a reader can put in a report.

Summing gives a single describable statement — Manning applied once to the whole wetted area —
that is wrong in a direction anyone can reason about. Lumping the full perimeter into one
hydraulic radius understates conveyance, so it returns a **deeper** normal depth for a given
discharge. That is the conservative direction for freeboard, and it is the same tie-breaker
[0003](./0003-shallowest-root-conveyance-not-monotone.md) used in choosing the shallowest root.

## Why suppress Froude rather than report it

`Dh = A/T` across two disjoint surfaces is a mean depth of two flows that are not hydraulically
the same flow, and a Froude number built on it describes neither. The section properties are
still correct — area, perimeter and mean velocity are honest sums — and it is the free-surface
quantities that stop meaning anything.

That is exactly the line [0004](./0004-closed-sections-error-at-surcharge.md) drew for **nearly
closed**, where the surface narrows toward the crown: properties survive, free-surface quantities
do not, and the condition is named rather than the result withheld. Disjoint regions get the same
treatment for the same reason, which keeps one rule rather than two.

## Consequences

- **`outline` changes shape for every section type, including the two that ship.** That is the
  cost of the decision and it is paid once, as `[[...]]` in place of `[...]`. `drawSection`'s
  bounding-box scan, its water polygon, its surface line, and its `invert0` and `bank` reads all
  iterate regions instead of points.
- **The merge elevation is a breakpoint and must be forced into the grid**, per 0003. It is where
  `P` drops discontinuously — two regions' perimeters become one and the two inner banks are lost
  — so `Q(y)` is discontinuous there in the same way as the bench in 0003's compound example.
  This is a second class of breakpoint alongside the bench edges already named there, and it is
  not visible in the input data as an elevation the user typed.
- **The region count is a function of depth and cannot be computed once.** Two low points at
  different elevations means the second region appears partway up, so the count changes in both
  directions as depth rises. `sectionProps` is already called per depth, which is the shape that
  makes this safe; anything caching a region count across a solve is wrong.
- **`containing` and `depthLimit` are both the top of the data**, and an arbitrary section is open
  there the way a trapezoid is open at `H`. Overtopping stays a warning. Nothing in 0004 applies
  — there is no crown, and no surcharge state to reach.
- **`y` is measured from the lowest elevation in the data**, so the invert is a property of the
  points rather than a datum the user sets. A section whose two low points sit at different
  elevations has an invert at one of them and a second region that begins above it.
