# mannings

Manning's equation solver for symmetric trapezoidal channels. One of a planned family of
small single-purpose environmental-engineering web tools linked from
[briannei.com](https://briannei.com); sibling repos are `briannei-site` (the landing page)
and `what-utm`.

## Shape of the project

**One file, no build, no dependencies.** Everything — markup, CSS, JS — lives in
`index.html`. This is deliberate, not laziness:

- Deploys to Cloudflare Pages as a direct upload with no build command. Every request is a
  static asset, so nothing touches the metered Workers/Functions quota.
- Runs from `file://` and offline, which matters for a calculation that may need to be
  reproducible years later.
- Auditable in one screen, which matters for a number that ends up in a report.

Do not introduce a build step, a framework, or a CDN dependency without a concrete reason
that these three properties don't cover. If a future phase genuinely needs one, say so
explicitly rather than absorbing it.

**The pure/impure seam.** The hydraulics functions at the top of the `<script>` block
(`section`, `discharge`, `normalDepth`, `regime`, `fmt`, `validate`) touch no DOM, and neither
do the sweep functions that follow them (`sweepable`, `sweepRange`, `validateRange`,
`sweepPoint`, `sweep`). Keep it that way — when these tools grow into a shared library the
seam is already cut. Everything below the `---- UI ----` comment may touch the DOM.

`sweep()` returns samples carrying **both** the response and the overtopping threshold from
one pass, so `drawSweep` does no hydraulics at all — it only maps numbers to pixels, exactly
as `drawSection` does. That is what stops the shaded boundary and the curve being computed
from different assumptions. Don't move a `discharge()` call below the seam to save an array
field.

**Terminology lives in [CONTEXT.md](./CONTEXT.md)** — a glossary, no implementation detail.
Architectural decisions live in [docs/adr/](./docs/adr/). Two exist: the sweep is
deliberately not an optimizer, and the sweep's response is the solve mode rather than a
separate control. Read those before adding to the sweep.

## Hard constraints

**No scrolling on a 1080p monitor.** This is a stated requirement, not a preference. It is
enforced structurally rather than by tuning: `body { height: 100dvh; overflow: hidden }`
with a CSS grid whose middle row is `minmax(0, 1fr)`, and the two figure SVGs absorb all
leftover space between them. `dvh` not `vh`, so mobile browser chrome can't push content out
of view. Below `68rem` the layout stacks and page scrolling is allowed — the guarantee is a
desktop promise, not a reason to break phones.

Verified at 1920×1080: `scrollWidth`/`scrollHeight` both exactly 1920/1080, no panel clipped,
`.figwrap` 1184×404 each with the sweep shown and 1184×835 with it hidden. Also checked at
1088px (the breakpoint) and 390px — no horizontal scroll at either, in either sweep state.

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
- **`placeRangeRow` must not re-insert the range row when it is already in place.**
  `insertAdjacentElement` re-inserts a node even when the position is unchanged, and
  re-inserting blurs whatever inside it has focus. Since `compute()` runs on every keystroke,
  that cost a focused range box its focus after every single digit.
- The figure column holds **two** figures in one panel: cross-section above, sweep chart
  below, split by a hairline on the second heading row rather than a divider element. Each
  figure travels with its own caption inside a `.half` at `flex: 1 1 0`, so the two take equal
  halves regardless of content — a caption inside one `.half` and outside the other made the
  drawing areas differ by a line of text. `flex-basis: auto` would let contents decide and
  `height` in the stacked media query would lose to a `0` basis, which is why that query
  resets `flex` to `0 0 auto`.
- The sweep half is **collapsible and off by default** (the "Show" checkbox on the sweep
  heading row). With it off the cross-section takes the whole panel. The toggle sits on that
  heading rather than in the controls column: it governs the figure, and a 21rem column has no
  width to spare. When off, the sweep radios are `disabled` — not hidden, so the field rows
  don't reflow — and the range row and swept-field highlight are hidden. The swept parameter
  and its range are still tracked while off, so switching back on restores the same chart
  rather than a re-seeded one.
- One panel, not two, so the vertical budget isn't spent twice on borders and padding.
- The sweep radio shares a row with the **input**, not the label. Putting it in the label row
  cost enough width to truncate "b — bottom width, m" to an ellipsis; a 21rem column has room
  for a full label or a radio beside it, not both. `.fctl` reserves the radio's column even on
  `H`, which has no radio, so every input box stays 125px. Related: `.field input` is scoped to
  `[type="number"]`, or the radio inherits the text box's padding, border and background.

**SI units only.** `k = 1.0`, so there is no unit conversion anywhere in the code. That is
the point: unit conversion is the most likely source of a wrong answer, and this removes
the category entirely. Adding US customary units means adding the `1.486` factor and a
label layer, and the factor and labels must not be able to disagree.

**Never vertically exaggerate the cross-section.** A channel section exists to show the
side slopes; distorting the vertical scale misrepresents `z`. Long profiles get exaggerated
with an annotation, cross-sections do not. A wide shallow channel legitimately leaves
whitespace in a tall panel — that is geometry, not a bug to fix. Halving the panel to make
room for the sweep chart made that whitespace more prominent, not less legitimate.

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

## Conventions in the math

- `z` is the side slope as **horizontal run per unit rise (H:V)**. This is the notorious
  ambiguity in the domain; the label says H:V on screen and should stay that way.
- `b = 0` (triangular) and `z = 0` (rectangular) are valid and must keep working. Only both
  zero is degenerate.
- `S₀ <= 0` is rejected with an explanation, never rendered as `NaN` — Manning is undefined
  for a flat or adverse bed.
- Normal depth uses **bisection, not Newton**. `Q(y)` is strictly increasing for a
  trapezoid, so a sign change brackets a unique root and the iteration cannot diverge. No
  derivative to get wrong and no failure mode to explain to a reviewer. The iteration cost
  is irrelevant. Don't "optimise" this.
- Froude in `0.95 < Fr < 1.05` is reported as **near critical**, not pinned to `Fr = 1.000`.
  Uniform-flow assumptions are unreliable there and exact criticality would be false
  precision.
- Results are 4 significant figures via `fmt()`.
- **`validateRange` is a near-sibling of `validate`, not a clone.** As a *range endpoint*,
  `Q = 0` and `y = 0` are well defined — both give exactly zero, and `y = 0` is the natural
  left edge of a rating curve — whereas as *point inputs* they are rejected, because a channel
  with no water is not a result anyone asked for. Do not "fix" the two to agree.
- A degenerate sweep sample yields `null`, never `NaN`, and leaves a gap in the curve rather
  than a straight line across territory with no solution. Nothing non-finite may reach the SVG.
- The sweep range is seeded per parameter and re-seeded only when the swept parameter or the
  mode changes — deliberately **not** when the other inputs change. An auto-range that tracked
  the current value would rescale the axis on every keystroke, so two screenshots taken a
  minute apart wouldn't be comparable.

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
by copying the JS — and comparing. For the sweep, dump samples from the live page
(`sweep(param, lo, hi, v, mode, n)` is global) as JSON and compare columns; the last run was
122 checks across 7 sweep configurations with 0 failures. Scratch scripts are throwaway;
recreate them as needed.

Known-good reference values:

| Inputs | Expected |
|---|---|
| `b=3, z=2, n=0.025, S=0.001, y=1.0` | `Q = 4.8385`, `A = 5`, `P = 7.4721`, `R = 0.66915`, `V = 0.96771`, `Fr = 0.36563` |
| `b=2, z=2, n=0.03, S=0.002, Q=5` | `y = 1.073169`, `A = 4.4497`, `P = 6.7994`, `R = 0.65443`, `V = 1.12367`, `Fr = 0.42671` |
| ↳ same channel, `H = 1.5` | capacity `10.1207`, freeboard `0.4268` |
| Triangular `b=0, z=2, y=1, n=0.03, S=0.002` | `Q = 1.7435` |
| Rectangular `b=2, z=0, y=1, n=0.03, S=0.002` | `Q = 1.8782` |

Sweep reference values, same channel (`b=2, z=2, H=1.5, n=0.03, S=0.002`):

| Sweep | Expected |
|---|---|
| depth mode, `Q` = 1 | `y = 0.4706924058` |
| depth mode, `Q` = 10 | `y = 1.4916624794` |
| depth mode, `n` = 0.01 (`Q`=5) | `y = 0.6170807542` |
| depth mode, `S₀` = 0.01 (`Q`=5) | `y = 0.7183463649` |
| depth mode, `b` = 0 (`Q`=5) | `y = 1.4844859793` |
| flow mode, `y` = 0.75 | `Q = 2.4330525968` |
| flow mode, `z` = 3 with `b` = 0, `y` = 1 | `Q = 2.7200434230` |
| flow mode, `b` = 3 with `z` = 0, `y` = 1 | `Q = 3.1813820870` |

Round-trip check: `normalDepth(discharge(y))` recovers `y = 1.234` to nine decimals.

Edge cases that must stay covered: overtopping warning when `y > H`, error state hides the
results block *and* any stale warning, blank field shows a prompt rather than garbage,
near-critical band, no horizontal page scroll at mobile widths. For the sweep: `min >= max`
and a zero lower bound on `n` or `S₀` each show an explanation *in place of the chart* while
the point results stay on screen; an invalid point input replaces the chart with a pointer to
the results panel rather than an unexplained blank box; a mode switch re-seeds the range and
moves the radio when the swept parameter becomes illegal, whether the sweep is shown or not;
`z = 0` with `b` swept from zero leaves a one-sample gap rather than a `NaN`; the operating
point falling outside the range is said so in the caption; typing several digits into a range
box keeps focus throughout. Link handling: no `sweep`/`smin`/`smax` keys falls back to the
default sweep; `swon` decides whether the chart is shown, and a link predating the toggle that
names a sweep parameter or range asked for a chart, so it gets one.

## Open items

- No remote deployment yet. Needs a Cloudflare Pages project (no build command, output =
  repo root) bound to `mannings.briannei.com`. Note `.claude/launch.json` now exists in the
  repo — if output is the repo root, either exclude `.claude/` from the upload or accept that
  a dev-tooling file gets published.
- The `briannei-site` landing page does not yet have a card for this tool.
- The sweep is static by design: no hover readout, no click-to-set-the-operating-point. Both
  were considered and deferred so that a screenshot is the complete figure and there is no
  pointer-versus-touch behaviour to maintain. Revisit only with a reason that beats those two.

## Deliberately out of scope for Phase 1

Circular, rectangular-as-its-own-mode, compound and natural sections; critical depth; US
customary units; CSV input; composite roughness; an `n` lookup table.

Also out: **inverse solve** ("what `b` gives `y` = 1.2?") and **optimization** ("what section
minimises excavation?"). The sweep is the substrate for both and forecloses neither, but it
deliberately reports a response rather than a recommendation — including *not* reporting the
parameter value where the curve crosses the bank, which is the tempting first step onto that
path. See `docs/adr/0001`.

## Repo hygiene

`main` tracks `origin/main` at `github.com/bnei/mannings`. This repo contains only its own
files — never write into a sibling repo to work around tooling.
