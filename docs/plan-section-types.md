# Plan — section types (trapezoidal, circular, arbitrary)

Agreed in a design session on 2026-07-30 and **completed on 2026-07-30** — all eight steps
shipped. Kept for the decisions ledger and the rejected alternatives below, which are the part
worth re-reading; the step list is history. Where this plan and `CLAUDE.md` disagree about what
the code does, `CLAUDE.md` is current and this is what was intended.

Two departures from the plan as written, both recorded in `CLAUDE.md`: the sweep range is also
re-seeded on a section change (the plan only listed swept parameter and mode), and the cost of
the new solver came out at 3.2× rather than the ~2× estimated below.

Goal: generalise the solver so section types are pluggable, ship **trapezoidal + circular**,
and leave a seam that an **arbitrary** station/elevation section fits without rework.

## Facts that drove the design

Two numerical checks, both reproducible:

Circular, `D = 1 m`, `n = 0.03`, `S₀ = 0.01`:

```
Qmax  = 1.117606 m³/s at y/D = 0.9380
Qfull = 1.038952 m³/s          (92.96% of Qmax)
```

So any `Q` in `(Qfull, Qmax)` has **two** normal depths. Arbitrary is worse — a 1 m trench
opening onto a 20 m bench at elevation 1.0 (`n = 0.03`, `S₀ = 0.01`):

```
y = 0.999   Q = 1.6005
y = 1.001   Q = 0.4388      wetted perimeter jumps 19 m, area does not
y = 1.100   Q = 2.6334
```

`Q ≈ 1.5` has three roots. **The real dichotomy is open vs closed/compound sections, not
trapezoid vs everything else** — design the seam around that.

Also found: `sweepPoint` (`index.html`, ~line 680) rebuilds its working object from a hardcoded
object literal, so any new geometry key is silently dropped rather than erroring. Latent bug,
fix independently.

## Decisions

| # | Decision |
|---|---|
| 1 | Seam accommodates arbitrary; only trapezoidal + circular ship this round |
| 2 | `normalDepth` enumerates all roots, reports the shallowest, flags that others exist — no monotonicity assumption, one code path for all section types |
| 3 | `H` is derived for closed sections; its input field disappears |
| 4 | Capacity stays "discharge at the containing dimension"; maximum discharge is a caveat on it, not a replacement |
| 5 | At or above the crown: error, no results. Pressurized flow is out of scope |
| 6 | Family term is **section type**; `section()` is renamed `sectionProps()` |
| 7 | One file. **No ES modules** — they fail from `file://` on CORS grounds, silently, and verification runs over HTTP so it would not surface |
| 8 | One registry per section type with namespaced `math` / `view` halves; hydraulics never reads `.view` (greppable) |
| 9 | `math.outline` holds the boundary polyline; section properties are **always analytic and never derived from it** — for arbitrary the two coincide, for circular they must not |
| 10 | Three declared parameter domains: `positive` (`n`, `S₀`, `D`, `H`), `nonNegative` (`b`, `z`), `positiveAsPoint` (`Q`, `y` — zero illegal as input, legal as range endpoint). Cross-parameter rules stay in the section type's own validate |
| 11 | All state stays in the URL fragment, uncompressed. New `sec` key; absent means trapezoidal, so every existing link works. Unknown parameter keys are ignored, not errors. Arbitrary geometry encodes as plain pairs: `pts=0,3;0.5,0;2,0;2.5,3` |
| 12 | Section type selector is a dropdown inside the `h1` ("Manning's equation — ⌄ trapezoidal channel"). Subtitle and `document.title` follow it. Needs a visible dropdown affordance |
| 13 | Froude is reported for closed sections but carries a **nearly closed** condition above `y/D = 0.9`, sibling to near critical |
| 14 | Circular verified by independent Python **plus** a published partial-flow ratio spot-check — an independent reimplementation of a wrong derivation agrees with itself |
| 15 | Freeboard row becomes **Depth to crown** for closed sections; row count stays identical across section types |
| 16 | ADRs 0003 and 0004 only. The registry shape and the `file://` constraint go into CLAUDE.md as constraints |

Rejected and worth not revisiting: bracketing on the rising limb only (correct for a pipe,
wrong for a compound section — its first local maximum is the top of the low-flow trench);
two parallel registries (drift); computing area from the drawing polygon (degrades wetted
perimeter for curved boundaries).

## Steps

**0 · Records.** `docs/adr/0003-shallowest-root-conveyance-not-monotone.md` and
`docs/adr/0004-closed-sections-error-at-surcharge.md`. 0003 matters most: it contradicts a
rationale currently stated in `index.html`, `README.md` and `CLAUDE.md`.

**1 · Fix the `sweepPoint` literal.** Independent of everything else.

**2 · Extract the generic core.** `manning(A, R, n, S)`; rename `section()` → `sectionProps()`;
split `validate` into generic and geometry halves. No behaviour change.

**3 · Introduce the registry, trapezoidal as the only member.** `math`/`view` halves, declared
parameter domains, `math.outline`. Rewire `drawSection` to consume `outline`; replace `KEYS`,
the `AXIS` table and `sweepRange`'s switch with declared params. No behaviour change.

**4 · Replace the solver.** Grid scan over `[0, ceiling]`, bisect each sign-change bracket,
report shallowest, flag multiplicity. Trapezoid is the only unbounded section, so the existing
doubling loop survives as the way to find a finite ceiling for open sections. Trapezoid yields
exactly one bracket, so every reference value and the round-trip check must come out identical.

**5 · UI, still trapezoid-only.** Dropdown in the `h1`, dynamic fields grid, dynamic subtitle
and `document.title`, `sec` in the fragment with back-compat. Re-verify the no-scroll matrix.

**6 · Add circular.** Arc geometry, `depthLimit = D`, surcharge error path, capacity-at-crown
plus maximum-discharge caveat, depth-to-crown row, nearly-closed condition, 96-segment outline
(drawing only), `D` sweepable with seed `[0.15, 2·D]`.

**7 · Verify circular.** Python plus published ratio check; new reference rows including
`y/D = 0.938` and the two-root band; sweep sample dump; layout at three widths.

**8 · Docs.** README method block and monotonicity paragraph; CLAUDE.md reference tables, the
registry rule, the `file://` constraint, the widened verification matrix.

Steps 1–4 change no behaviour and are checkable against the existing reference values.
Behaviour first changes at step 5. **Stop for review after step 4** — it is the last point
where the existing reference values still prove everything.

## Risks

- **Grid resolution is now a correctness parameter.** A root pair narrower than the grid
  spacing is missed silently. Fine for circular (the pair spans ~6% of depth). When arbitrary
  lands, every breakpoint elevation must be forced into the grid — that is where the
  discontinuities are. Write this into CLAUDE.md at step 4, not step 7.
- **Sweep cost roughly doubles.** `compute()` runs on every keystroke and `drawCurve()`
  re-samples up to 400 points, each doing a full root enumeration in depth mode: ~240
  `discharge()` calls per sample today, ~400 after. Should stay imperceptible — measure at
  step 4 rather than assume.
- **Step 5 is where the hard constraint can break.** A `<select>` in the `h1` changes titlebar
  height in every layout state and the fields grid becomes dynamic. Isolated from the circular
  work for exactly this reason.

## Verification matrix

Trapezoidal is the worst case for controls-column height (circular replaces `b`, `z`, `H` with
`D` alone, freeing a row), so: full six-state matrix on trapezoidal (1920×1080, 1088px, 390px ×
sweep shown/hidden), plus a confirming pass for circular at the three widths. Nine states, not
eighteen. Serve over HTTP per CLAUDE.md — `file://` snapshots are unreliable.

## Explicitly out of scope this round

Arbitrary sections (seam only), the arbitrary input widget, composite roughness, US customary
units, critical depth, inverse solve and optimization (see `docs/adr/0001`).
