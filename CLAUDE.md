# mannings

Manning's equation solver for open channels and closed conduits — trapezoidal and circular
section types ship. One of a planned family of
small single-purpose environmental-engineering web tools linked from
[briannei.com](https://briannei.com); sibling repos are `briannei-site` (the landing page)
and `what-utm`.

## Shape of the project

**One file, no build, no dependencies.** Everything — markup, CSS, JS — lives in
`index.html`. This is deliberate, not laziness:

- Deployed on Cloudflare Pages with no build command. Every request is a static asset, so
  nothing touches the metered Workers/Functions quota.
- Runs from `file://` and offline, which matters for a calculation that may need to be
  reproducible years later. Every path in the file is relative for that reason — a root-absolute
  `/favicon.svg` resolves to the filesystem root from `file://` and 404s in exactly the offline
  case the shape exists to serve.
- Auditable in one screen, which matters for a number that ends up in a report.

Do not introduce a build step, a framework, or a CDN dependency without a concrete reason
that these three properties don't cover. If a future phase genuinely needs one, say so
explicitly rather than absorbing it.

**The pure/impure seam.** The hydraulics functions at the top of the `<script>` block
(`manning`, `sectionProps`, `discharge`, `normalDepths`, `normalDepth`, `maxDischarge`,
`regime`, `fmt`, `validate`) touch no DOM, and neither do the sweep functions that follow them
(`sweepable`, `sweepRange`, `validateRange`, `sweepPoint`, `sweep`). Keep it that way — when
these tools grow into a shared library the seam is already cut. Everything below the
`---- UI ----` comment may touch the DOM.

`sweep()` returns samples carrying **both** the response and the exceedance threshold from
one pass, so `drawSweep` does no hydraulics at all — it only maps numbers to pixels, exactly
as `drawSection` does. That is what stops the shaded boundary and the curve being computed
from different assumptions. Don't move a `discharge()` call below the seam to save an array
field.

**One registry per section type**, in `SECTIONS`, with namespaced `math` and `view` halves.
The hydraulics never read `.view` — that rule is greppable, which is the point of the
namespacing. Two parallel tables keyed by section type would drift; one registry cannot.

A section type's `math` half owns `params`, `depthLimit`, `containing`, `props`, `validate`
and `outline`. Everything else is generic and must stay that way:

- **Every input is declared exactly once**, as a parameter object carrying `key`, `sym`,
  `desc`, `name`, `domain`, `sweep`, `def`, `step` and `seed`. That declaration is the only
  statement of what a section takes — the fields grid, the sweep axis label, the validation,
  the seeded range, the spinner increment, the URL keys and the aria labels are all generated
  from it. Do not add a hardcoded list of parameter names anywhere; several used to exist and
  each was a thing to keep in step.
- **`step` governs the arrow keys and nothing else.** `Q` steps by 1, `b` and `z` by 0.5, `H`
  and `y` by 0.1, `D` by 0.05 and `n` by 0.001, `S₀` by 0.01. A typed value off the step grid
  is still read and solved exactly as given, because `readInputs` parses `input.value` rather
  than asking the input whether it is valid — and nothing styles `:invalid`. Keep it that way:
  a step that could reject a hand-entered `n` would be a wrong answer, not a nicety.
- **The arrows cannot step a field to a value validation would then reject.** `stepFloor`
  reads the floor off the same `domain` that does the rejecting — `0` for `nonNegative`, one
  whole step for the two positive domains — so it cannot drift from the validation the way a
  hand-written `min` per parameter would. `stepCeiling` is the top end and the only cross-field
  one: a closed section's `y` in flow mode stops one step below the crown, because `y >= D`
  is an error (`docs/adr/0004`) and a `max` set at the crown would still let the spinner land
  exactly on it. It depends on `D`, so `compute()` re-sets it on every keystroke — safe
  because setting an attribute neither blurs nor moves the field.
  A trapezoid's `y` gets **no** ceiling: overtopping is a warning about a flow that really
  happens, so the arrows may walk past the bank. Two states remain reachable and should stay
  reachable, since neither is something an arrow did: `b` and `z` both arrowed to zero is the
  documented degenerate cross-section, and shrinking `D` below a standing `y` surcharges.
  The claim holds **while the crown clears one step of `y` (0.1 m)** — i.e. `D > 0.1`. Below
  that, `stepCeiling` returns `null` and the `max` comes off, so the arrows can reach `y = 0.1
  >= D`. It is not fixable at the ceiling alone: `stepFloor` already pins `min = 0.1` from the
  `positiveAsPoint` domain, and HTML clamps a `max` below `min` up to `min`, so on a sub-100 mm
  pipe *no* arrow-reachable `y` is valid. Typing still works and still validates correctly.
  Fixing it properly means a finer `step` for `y` or a geometry-aware floor; neither is worth
  the churn for sub-100 mm circular sections, so the boundary is documented instead.
- **Three parameter domains**: `positive` (`n`, `S₀`, `D`, `H`), `nonNegative` (`b`, `z`) and
  `positiveAsPoint` (`Q`, `y`). The third is what keeps `validate` and `validateRange`
  near-siblings by declaration rather than by two hand-synced lists.
- **`math.outline` is for drawing only.** Section properties are always analytic and never
  derived from it. For a trapezoid the two coincide; for a circle they must not — a polygon
  under-reports a curved perimeter, and `P` is two thirds of `R`. The circle's outline is
  96 segments, which is a drawing decision and nothing else.
- **No ES modules.** They fail from `file://` on CORS grounds, silently, and verification runs
  over HTTP, so the failure would not surface until someone opened the file offline — which is
  precisely the case the single-file shape exists to serve.

**Terminology lives in [CONTEXT.md](./CONTEXT.md)** — a glossary, no implementation detail.
Architectural decisions live in [docs/adr/](./docs/adr/). Four exist: the sweep is
deliberately not an optimizer (0001), the sweep's response is the solve mode rather than a
separate control (0002), normal depth enumerates roots and reports the shallowest (0003), and
a closed section at its crown is an error rather than a warning (0004). Read 0001 and 0002
before adding to the sweep, 0003 and 0004 before touching the solver or adding a section type.

## Hard constraints

**No scrolling on a 1080p monitor.** This is a stated requirement, not a preference. It is
enforced structurally rather than by tuning: `body { height: 100dvh; overflow: hidden }`
with a CSS grid whose middle row is `minmax(0, 1fr)`, and the two figure SVGs absorb all
leftover space between them. `dvh` not `vh`, so mobile browser chrome can't push content out
of view. Below `68rem` the layout stacks and page scrolling is allowed — the guarantee is a
desktop promise, not a reason to break phones.

Verified at 1920×1080 on **both** section types: `scrollWidth`/`scrollHeight` both exactly
1920/1080, no panel clipped, `.figwrap` 1184×404 each with the sweep shown and 1184×891 with
it hidden — hiding the sweep also hides the "Sensitivity sweep" subhead, so the cross-section
reclaims that row (~60px) on top of the chart's half. Also checked at 1088px (the breakpoint) and 390px — no horizontal scroll at either,
in either sweep state, in either section type. Trapezoidal is the worst case for the controls
column: circular replaces `b`, `z` and `H` with `D` alone, freeing a row.

The `<select>` in the `h1` costs the titlebar about 4px over the plain heading it replaced,
and the figure column absorbed it — a few px off each figure rather than a taller page, which
is the structural guarantee doing its job.

Consequences to respect:
- Every grid item needs explicit `grid-area`. Relying on source order already caused one
  bug where the results panel landed in a phantom fourth row.
- Every panel needs `min-height: 0` and `min-width: 0` or flex children refuse to shrink.
- `.metrics` has `overflow-y: auto` as a safety valve for short windows. At 1080p it never
  engages.
- Never set `display` inline on an element that also gets a `.hidden` class — inline styles
  out-specify the class and the element refuses to hide. This has bitten twice. The same trap
  in a stylesheet: `.hidden` is declared early, so any later single-class rule setting
  `display` beats it. `.sweeprange` and `.half` therefore carry explicit `.x.hidden` rules.
- The figure column holds **two** figures in one panel: cross-section above, sweep chart
  below, split by a hairline on the second heading row rather than a divider element. Each
  figure travels with its own caption inside a `.half` at `flex: 1 1 0`, so the two take equal
  halves regardless of content — a caption inside one `.half` and outside the other made the
  drawing areas differ by a line of text. `flex-basis: auto` would let contents decide and
  `height` in the stacked media query would lose to a `0` basis, which is why that query
  resets `flex` to `0 0 auto`.
- The sweep half is **collapsible and off by default** (a switch in the Inputs column, in
  `.sweep-callout` below the fields grid — not on the figure heading). With it off the
  cross-section takes the whole panel. The switch lives in the controls column rather than on
  the figure's heading row: that row has just the "Sensitivity sweep" `h2` now, and the 21rem
  controls column has room below the fields for the switch and its range row even though it
  never had width to spare for one beside a heading. The range row (`#sweeprange`) is a fixed
  child of `.sweep-callout`, not repositioned per swept parameter — there is no `placeRangeRow`
  any more; the row labels itself with the swept symbol instead of sitting next to that
  parameter's field. When off, the sweep radios are hidden via `opacity`/`visibility` on the
  radio itself (not `display`), so the `.fctl` column they sit in still reserves their width
  and the field rows don't reflow when they reappear. The range row and swept-field highlight
  are also hidden. The swept parameter and its range are still tracked while off, so switching
  back on restores the same chart rather than a re-seeded one.
- One panel, not two, so the vertical budget isn't spent twice on borders and padding.
- The sweep radio shares a row with the **input**, not the label. Putting it in the label row
  cost enough width to truncate "b — bottom width, m" to an ellipsis; a 21rem column has room
  for a full label or a radio beside it, not both. `.fctl` reserves the radio's column even on
  `H`, which has no radio, so every input box stays 125px. Related: `.field input` is scoped to
  `[type="number"]`, or the radio inherits the text box's padding, border and background.
- The section-type `<select>` lives in the `h1`. `appearance: none` removes the platform
  caret, so `.secpick::after` supplies one — without a caret, a select styled to match the
  heading around it reads as plain text and nobody finds it. The static `<title>` stays
  section-neutral because it is what a crawler, a link preview and the tab before first paint
  see; `applyIdentity()` specialises it per section at runtime.
- **The results and error blocks are live regions** — `aria-live="polite"` on `#results-body`,
  `role="alert"`/`aria-live="assertive"` on `#error`. Every output is rewritten in place on each
  keystroke, and an error additionally *hides* the results block, so without these a screen
  reader gets nothing where a sighted user sees the answer change.
- **The fields grid is generated, and the field elements are replaced on a section change.**
  Nothing may cache an input element, the sweep radios or `KEYS` across one; `setSection` is
  the only place that rebuilds them and it rebuilds all four together. Values are carried over
  for parameters the new section type shares — `Q`, `y`, `n`, `S₀` — because comparing a
  culvert with a swale at the same design flow is the point of having both.

**SI units only.** `k = 1.0`, so there is no unit conversion anywhere in the code. That is
the point: unit conversion is the most likely source of a wrong answer, and this removes
the category entirely. Adding US customary units means adding the `1.486` factor and a
label layer, and the factor and labels must not be able to disagree.

**Never vertically exaggerate the cross-section.** A channel section exists to show the
side slopes; distorting the vertical scale misrepresents `z`. Long profiles get exaggerated
with an annotation, cross-sections do not. A wide shallow channel legitimately leaves
whitespace in a tall panel — that is geometry, not a bug to fix. Halving the panel to make
room for the sweep chart made that whitespace more prominent, not less legitimate.

**The cross-section is dimensioned, not labelled.** `T`, `y` and the freeboard are drawn as
drafted dimensions — extension lines off the geometry, a dimension line between them, arrowheads
at its ends — rather than as text floating near the feature, because a number beside a shape
doesn't say what it measures and a dimension does. The helpers are `dimArrow`, `dimLine`,
`dimText`, `dimH` and `dimV`; the whole dimension takes one semantic color, so a flagged value
(overtopping) reads as a unit rather than as red digits. Text stays upright on a vertical
dimension — unidirectional dimensioning, the convention civil cross-sections use.

Three details are load-bearing and each replaced something that looked wrong:

- **Extension lines are faint (`DIM_FAINT`), the dimension line and arrows are not.** On a
  sloped bank the depth's two feature points are far apart in `x`, so one witness line is
  inevitably long; at full weight it reads as part of the channel. The fix is hierarchy, not
  moving the dimension somewhere it fits better.
- **The text breaks its line with a background-colored stroke under the glyphs**
  (`paint-order="stroke"`), not with an opaque rect. The rect had to be sized from a
  character-width guess, which is why the dimension font was once monospace; the halo needs no
  measurement, so the figure is back to one font. It assumes flat background behind the text —
  true for every current dimension, and the thing to check before placing one over the water.
- **A span under `DIM_TIGHT` puts its arrows outside pointing in**, with tails past them and the
  text moved clear. Reachable and common: a freeboard of 20 mm otherwise draws two arrowheads
  crossing under a label wider than the thing it measures.

`padX` in `drawSection` has a floor of 64 because the widest label (`y = 0.7183 m`) is centered
on a dimension line 18px outside the shape's bounding box. That floor is set by the label, not
by taste — lower it and the depth clips off a phone-width panel.

**Linear sweep axes only.** A log x-axis would read better for `S₀`, which spans 20× across
its seeded range, but a log axis cannot represent zero and `b = 0` (triangular) and `z = 0`
(rectangular) must stay sweepable. The seeded `S₀` range is kept narrow instead. Don't
"improve" this without solving the zero problem first.

**The sweep y-axis always starts at zero** and its top is set by the response, never by the
threshold. That is what makes the shaded exceedance region visible whenever overtopping occurs
somewhere in the range: an overtopping sample has response above threshold, so the boundary is
necessarily on-chart. A threshold far above every response is simply clipped away, which is
the correct picture of a channel with ample freeboard. A truncated axis would also be
misleading in a report figure.

**A closed section at its crown is an error, not a warning.** Overtopping keeps a free surface
and the numbers still describe uniform flow, so it is reported with a caveat. Surcharge does
not: flow becomes pressurized, `S₀` stops being the energy slope, and Manning's equation is no
longer the equation being asked about. The line is at `y >= D`, not `y > D` — flowing exactly
full already has a zero-width surface and a divergent hydraulic depth. See `docs/adr/0004`.

**Capacity is the discharge at the containing dimension**, at the crown for a closed section,
even though a circular section conveys about 7.6% more just below it. The maximum discharge is
reported as a caveat on capacity, never as the capacity: the peak is not a depth flow can be
held at, and a culvert sized to it is a culvert sized to surcharge.

## Conventions in the math

- `z` is the side slope as **horizontal run per unit rise (H:V)**. This is the notorious
  ambiguity in the domain; the label says H:V on screen and should stay that way.
- `b = 0` (triangular) and `z = 0` (rectangular) are valid and must keep working. Only both
  zero is degenerate.
- `S₀ <= 0` is rejected with an explanation, never rendered as `NaN` — Manning is undefined
  for a flat or adverse bed.
- Normal depth **enumerates every root and reports the shallowest**, stating that others
  exist when they do. `Q(y) - Q` is sampled on a grid over `[0, ceiling]` and every sign
  change is bisected. There is no monotonicity assumption anywhere — that argument holds for
  a trapezoid and fails for a circular section and for any compound one. See `docs/adr/0003`.
- **`GRID` is a correctness parameter, not a performance knob.** A root pair narrower than
  one grid cell is missed silently. At 200 the circular pair, spanning about 6% of the depth
  range, is found with two orders of magnitude to spare — an independent Python check at
  1000 finds the same two roots. When arbitrary sections land, **every breakpoint elevation
  must be forced into the grid**: that is where the discontinuities are, and a bench edge can
  sit anywhere between two evenly spaced samples.
- Still **bisection, not Newton**, and for the same reasons: a sign change brackets a root,
  there is no derivative to get wrong, and the iteration cannot diverge. Only the bracketing
  changed. The iteration cost is irrelevant. Don't "optimise" this.
- `discharge()` returns **`NaN` above the depth limit**, not zero and not the full-bore value,
  so a sweep past a crown leaves a gap rather than a curve running flat along the top of a
  pipe. `y == limit` is allowed, because capacity is by definition the discharge at the crown.
- Froude in `0.95 < Fr < 1.05` is reported as **near critical**, not pinned to `Fr = 1.000`.
  Uniform-flow assumptions are unreliable there and exact criticality would be false
  precision.
- Results are 4 significant figures via `fmt()`. A number reported as a **bound** rather than as
  a result — the maximum discharge, in both the no-normal-depth error and the peak-conveyance
  caveat — goes through `fmtFloor()`, which truncates toward zero at the same precision.
  Rounding to nearest can round a ceiling *up*, so `Qmax = 1.1176` would print as `1.118` and
  the tool would reject `Q = 1.1177` while naming a larger flow the section cannot carry.
  Overstating capacity is the wrong direction for a number that may size a culvert.
- **`validateRange` is a near-sibling of `validate`, not a clone.** As a *range endpoint*,
  `Q = 0` and `y = 0` are well defined — both give exactly zero, and `y = 0` is the natural
  left edge of a rating curve — whereas as *point inputs* they are rejected, because a channel
  with no water is not a result anyone asked for. Do not "fix" the two to agree.
- A degenerate sweep sample yields `null`, never `NaN`, and leaves a gap in the curve rather
  than a straight line across territory with no solution. Nothing non-finite may reach the SVG.
- The sweep range is seeded per parameter and re-seeded only when the swept parameter, the
  mode or the **section type** changes — deliberately **not** when the other inputs change. An
  auto-range that tracked the current value would rescale the axis on every keystroke, so two
  screenshots taken a minute apart wouldn't be comparable. A section change is exempt because
  it is a deliberate act rather than a keystroke, and because the span that makes sense for a
  depth or a width is set by the geometry that just changed.
- `y`'s seeded range comes from the **containing dimension**, not from `H`, so a closed
  section seeds from its crown without the seed needing to know which kind of section it is.

## Verification

Browser tooling is unreliable against `file://` URLs — navigation and reload both fail
intermittently, and a stale snapshot will happily report the *previous* version of the page,
which is worse than an error. **Serve over HTTP instead.** `.claude/launch.json` defines a
`static` config (`python -m http.server 8787`); start it and load
`http://127.0.0.1:8787/index.html`. That path has been reliable. Note that the page still
must work from `file://` — that constraint is unchanged, it just can't be *verified* that way.

Prefer asserting on DOM geometry over reading screenshots: the screenshot raster in this
environment renders the viewport at the wrong scale, and element rects are the stronger
evidence anyway.

The math is verified by **reimplementing it independently in Python** from the equations — not
by copying the JS — and comparing. Dump values from the live page as JSON and compare columns:
`sec`, `SECTIONS`, `sectionProps(sec, v, y)`, `discharge(sec, v, y)`, `normalDepths(sec, v, Q)`,
`maxDischarge(sec, v)` and `sweep(sec, param, lo, hi, v, mode, n)` are all global. Scratch
scripts are throwaway; recreate them as needed.

For the circular section, "independently" has to mean more than retyping the closed form —
an independent reimplementation of a wrong derivation agrees with itself. The last run got
`A` by Simpson quadrature of the width function (substituting `t = y·u²` to smooth the
square-root cusp at the invert) and `P` by a chord sum along the boundary, using **no**
arc-length or area formula: 159 checks agreeing to 5e-11 on `A`, 3e-9 on `P` and 2e-9 on `Q`,
plus 19 root-enumeration and sweep checks, 0 failures. It then checked the published
dimensionless anchors, which are the part that would catch a derivation wrong in a way both
implementations shared:

| Anchor | Computed | Published |
|---|---|---|
| `Q/Qfull` at `y/D` = 0.5 | 0.5000 | 0.500 |
| `Q/Qfull` at `y/D` = 0.8 | 0.9775 | 0.978 |
| peak `Q/Qfull` | 1.0757 | 1.076 |
| `y/D` at peak `Q` | 0.9382 | 0.938 |
| peak `V/Vfull` | 1.1400 | 1.14 |
| `y/D` at peak `V` | 0.8128 | 0.81 |

Known-good reference values:

| Inputs | Expected |
|---|---|
| `b=3, z=2, n=0.025, S=0.001, y=1.0` | `Q = 4.8385`, `A = 5`, `P = 7.4721`, `R = 0.66915`, `V = 0.96771`, `Fr = 0.36563` |
| `b=2, z=2, n=0.03, S=0.01, Q=5` | `y = 0.718346`, `A = 2.4687`, `P = 5.2125`, `R = 0.47361`, `V = 2.0253`, `Fr = 0.90868` |
| ↳ same channel, `H = 1.5` | capacity `22.631`, freeboard `0.78165` |
| Triangular `b=0, z=2, y=1, n=0.03, S=0.01` | `Q = 3.8987` |
| Rectangular `b=2, z=0, y=1, n=0.03, S=0.01` | `Q = 4.1997` |

That second row is the page as it loads — the input defaults are exactly that channel, so
it can be checked without typing anything. Note `Fr = 0.9087` is a fast subcritical flow,
close to but outside the near-critical band; a change to the defaults that pushes it past
`0.95` would put the landing state into the unreliable-results warning.

Sweep reference values, same channel (`b=2, z=2, H=1.5, n=0.03, S=0.01`):

| Sweep | Expected |
|---|---|
| depth mode, `Q` = 1 | `y = 0.3024374329` |
| depth mode, `Q` = 10 | `y = 1.0163579858` |
| depth mode, `Q` = 22 | `y = 1.4804252636` |
| depth mode, `n` = 0.01 (`Q`=5) | `y = 0.4013178935` |
| depth mode, `S₀` = 0.05 (`Q`=5) | `y = 0.4706924058` |
| depth mode, `b` = 0 (`Q`=5) | `y = 1.0977898835` |
| flow mode, `y` = 0.75 | `Q = 5.4404709993` |
| flow mode, `z` = 3 with `b` = 0, `y` = 1 | `Q = 6.0822019956` |
| flow mode, `b` = 3 with `z` = 0, `y` = 1 | `Q = 7.1137866090` |

The `Q = 22` row is there to keep a sample just under the bank now that the steeper bed puts
`Q = 10` well below it — the near-overtopping path needs a reference value too. The `S₀` row
is swept to 0.05 rather than 0.01 because 0.01 is now the operating point itself, which would
make it a check against the point row rather than against the sweep.

Round-trip check: `normalDepth(discharge(y))` recovers `y = 1.234` to nine decimals.

Circular reference values (`D = 1, n = 0.03, S₀ = 0.01` throughout — `Q` scales as `D^(8/3)`,
so the ratios in the anchor table above hold at any diameter):

| Inputs | Expected |
|---|---|
| flow mode, `y = 0.5` (half full) | `Q = 0.5194757795`, `A = 0.39270`, `P = 1.5708`, `R = 0.25`, `T = 1`, `V = 1.3228`, `Fr = 0.67409` |
| flowing full, `y = D` | capacity `1.0389515590` — exactly twice the half-full discharge |
| maximum discharge | `1.1176065602` at `y = 0.9381812134` |
| depth mode, `Q = 0.5` | `y = 0.4889235197` |
| depth mode, `Q = 1.08` | **two** roots, `0.8604662480` and `0.9912168728`; the first is reported |
| depth mode, `Q = 1.15` | no normal depth — above the maximum discharge |
| depth mode sweep, `D = 1.075` (`Q` = 0.5) | `y = 0.4707320872` |
| flow mode sweep, `D = 1.075` (`y` = 0.5) | `Q = 0.5561711779`, threshold `1.2599438870` |

The `Q = 1.08` row is the whole reason for `docs/adr/0003` and must keep returning two roots.
The `y = 0.5` row is worth keeping for a different reason: a half-full circular section
carries **exactly half** its full-bore discharge, because `R` is `D/4` at both depths. That is
a closed-form identity, so it catches an error in `A` or `P` that a numerical check might not.

Edge cases that must stay covered: overtopping warning when `y > H`, error state hides the
results block *and* any stale warning, blank field shows a prompt rather than garbage,
near-critical band, no horizontal page scroll at mobile widths. For the sweep: `min >= max`
and a zero lower bound on `n` or `S₀` each show an explanation *in place of the chart* while
the point results stay on screen; an invalid point input replaces the chart with a pointer to
the results panel rather than an unexplained blank box; a mode switch re-seeds the range and
moves the radio when the swept parameter becomes illegal, whether the sweep is shown or not;
`z = 0` with `b` swept from zero leaves a one-sample gap rather than a `NaN`; the operating
point falling outside the range is said so in the caption; typing several digits into a range
box keeps focus throughout.

For closed sections: `y >= D` in flow mode is an error with the results block hidden, not a
warning; a discharge above the maximum says what the maximum is and at what depth, rather than
telling the user to check their geometry; `y > 0.9·D` raises the nearly-closed condition; the
capacity caveat naming the peak is always present; a sweep of `y` past the crown leaves gaps
rather than a flat line; the results panel has the same nine rows as it does for a trapezoid,
with `Capacity at crown` and `Depth to crown` in place of `Capacity at H` and `Freeboard`.

For the dimensions, assert on `getBBox` rather than on a screenshot: no label may fall outside
the viewBox and no two may overlap, checked on both section types at 1920, 1088 and 390 px and
in both sweep states. The four geometries that exercise the awkward paths are a near-zero
freeboard (`y = 1.48` of `H = 1.5`), a near-zero depth (`y = 0.03`), a circular section just
under its crown (`y = 0.985·D`), and overtopping, where the depth dimension turns red and the
freeboard is not drawn at all. The narrow-panel case is the one that catches a `padX` change.

Link handling: no `sweep`/`smin`/`smax` keys falls back to the default sweep; `swon` decides
whether the chart is shown, and a link predating the toggle that names a sweep parameter or
range asked for a chart, so it gets one. No `sec` key means trapezoidal, so every link shared
before section types existed still opens; unknown keys are ignored rather than treated as
errors, so a link naming a section type this build doesn't have still opens on what it does
understand. That includes names inherited from `Object.prototype` — `#sec=constructor` and
`#sec=toString` must fall back like `#sec=egg` does, which is why the lookup goes through
`hasOwnProperty` rather than a bare `SECTIONS[h.sec] ||` read. `mode` is applied only if it
names a real radio: any other value would uncheck both and render the group as an unanswered
choice while `currentMode()` silently carried on in depth mode.

## Open items

- **[docs/plan-section-types.md](./docs/plan-section-types.md) is complete** — all eight steps.
  The plan is kept for its decisions ledger and its rejected alternatives, not as a work list.
- **Arbitrary (station/elevation) sections are designed for but not built.** The seam is cut
  and this is the shape it expects: a `params` list holding the point pairs, `outline`
  returning them directly, `props` integrating them analytically, and `depthLimit` at the top
  of the data. The one thing that will *not* work as-is is the grid — see the `GRID` note
  above, every breakpoint elevation has to be forced into it. The fragment encoding was agreed
  as plain pairs, `pts=0,3;0.5,0;2,0;2.5,3`.
- The `briannei-site` landing page does not yet have a card for this tool.
- The sweep is static by design: no hover readout, no click-to-set-the-operating-point. Both
  were considered and deferred so that a screenshot is the complete figure and there is no
  pointer-versus-touch behaviour to maintain. Revisit only with a reason that beats those two.

## Deliberately out of scope

Arbitrary, compound and natural sections; rectangular as its own section type (it is `z = 0`);
pressurized flow (`docs/adr/0004`); critical depth; US customary units; CSV input; composite
roughness; an `n` lookup table.

Also out: **inverse solve** ("what `b` gives `y` = 1.2?") and **optimization** ("what section
minimises excavation?"). The sweep is the substrate for both and forecloses neither, but it
deliberately reports a response rather than a recommendation — including *not* reporting the
parameter value where the curve crosses the bank, which is the tempting first step onto that
path. See `docs/adr/0001`.

## Repo hygiene

`main` tracks `origin/main` at `github.com/bnei/mannings`. This repo contains only its own
files — never write into a sibling repo to work around tooling.
