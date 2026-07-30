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
(`section`, `discharge`, `normalDepth`, `regime`, `fmt`, `validate`) touch no DOM. Keep it
that way — when these tools grow into a shared library the seam is already cut. Everything
below the `---- UI ----` comment may touch the DOM.

## Hard constraints

**No scrolling on a 1080p monitor.** This is a stated requirement, not a preference. It is
enforced structurally rather than by tuning: `body { height: 100dvh; overflow: hidden }`
with a CSS grid whose middle row is `minmax(0, 1fr)`, and the cross-section SVG absorbs all
leftover space. `dvh` not `vh`, so mobile browser chrome can't push content out of view.
Below `68rem` the layout stacks and page scrolling is allowed — the guarantee is a desktop
promise, not a reason to break phones.

Consequences to respect:
- Every grid item needs explicit `grid-area`. Relying on source order already caused one
  bug where the results panel landed in a phantom fourth row.
- Every panel needs `min-height: 0` and `min-width: 0` or flex children refuse to shrink.
- `.metrics` has `overflow-y: auto` as a safety valve for short windows. At 1080p it never
  engages.
- Never set `display` inline on an element that also gets a `.hidden` class — inline styles
  out-specify the class and the element refuses to hide. This has bitten twice.

**SI units only.** `k = 1.0`, so there is no unit conversion anywhere in the code. That is
the point: unit conversion is the most likely source of a wrong answer, and this removes
the category entirely. Adding US customary units means adding the `1.486` factor and a
label layer, and the factor and labels must not be able to disagree.

**Never vertically exaggerate the cross-section.** A channel section exists to show the
side slopes; distorting the vertical scale misrepresents `z`. Long profiles get exaggerated
with an annotation, cross-sections do not. A wide shallow channel legitimately leaves
whitespace in a tall panel — that is geometry, not a bug to fix.

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

## Verification

Browser tooling in the dev environment has been unreliable for local files. The math is
verified by **reimplementing it independently in Python** from the equations — not by
copying the JS — and comparing. Scratch scripts are throwaway; recreate them as needed.

Known-good reference values:

| Inputs | Expected |
|---|---|
| `b=3, z=2, n=0.025, S=0.001, y=1.0` | `Q = 4.8385`, `A = 5`, `P = 7.4721`, `R = 0.66915`, `V = 0.96771`, `Fr = 0.36563` |
| `b=2, z=2, n=0.03, S=0.002, Q=5` | `y = 1.073169`, `A = 4.4497`, `P = 6.7994`, `R = 0.65443`, `V = 1.12367`, `Fr = 0.42671` |
| ↳ same channel, `H = 1.5` | capacity `10.1207`, freeboard `0.4268` |
| Triangular `b=0, z=2, y=1, n=0.03, S=0.002` | `Q = 1.7435` |
| Rectangular `b=2, z=0, y=1, n=0.03, S=0.002` | `Q = 1.8782` |

Round-trip check: `normalDepth(discharge(y))` recovers `y = 1.234` to nine decimals.

Edge cases that must stay covered: overtopping warning when `y > H`, error state hides the
results block *and* any stale warning, blank field shows a prompt rather than garbage,
near-critical band, no horizontal page scroll at mobile widths.

## Open items

- **The three-column layout has never been visually verified** — it was built while browser
  tooling was down. Check for clipping and overflow at 1920×1080 before trusting it.
- The cross-section uses only ~25% of its panel height at the default geometry. Candidate
  use for the space: a small **Q vs y curve** with the operating point marked, which would
  also show the monotonic `Q(y)` the bisection relies on. Not scoped yet.
- No remote deployment yet. Needs a Cloudflare Pages project (no build command, output =
  repo root) bound to `mannings.briannei.com`.
- The `briannei-site` landing page does not yet have a card for this tool.

## Deliberately out of scope for Phase 1

Circular, rectangular-as-its-own-mode, compound and natural sections; critical depth; US
customary units; CSV input; composite roughness; an `n` lookup table.

## Repo hygiene

`main` tracks `origin/main` at `github.com/bnei/mannings`. This repo contains only its own
files — never write into a sibling repo to work around tooling.
