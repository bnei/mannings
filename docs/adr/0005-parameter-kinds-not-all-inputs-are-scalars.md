# A parameter declares its kind; the generic layers never branch on section type

Every parameter declared so far is implicitly one thing: a finite float. The declaration carries
`key`, `sym`, `desc`, `name`, `domain`, `sweep`, `def`, `step` and `seed`, and every one of those
fields is scalar-shaped — `domain` names one of three float predicates, `step` is a spinner
increment, `seed` returns a numeric `[lo, hi]`, and `buildFields` renders an
`<input type="number">`. That is true of all eight parameters that exist (`Q`, `y`, `b`, `z`,
`H`, `n`, `S₀`, `D`) and is written down nowhere, because until now nothing contradicted it.

An arbitrary section's geometry is a variable-length list of station/elevation pairs. It has no
step, no numeric domain, no seeded range, no spinner, and it cannot be swept. Admitting it means
the generic layers have to branch on **what kind of value a parameter holds**. The decision is
that they branch on a declared `kind`, and never on the section type.

`kind: "scalar"` is the default and its absence means scalar, so no existing declaration changes
and this ADR's diff is additive.

## Alternatives rejected

**Several scalar parameters — `x1`, `y1`, `x2`, `y2`, …** Fails on the first requirement: the
point count is variable, and that variability is the entire content of the word "arbitrary".
`paramList` builds a fixed array from `sec.math.params`, so a section that redeclared its
parameter list as the user added a point would rebuild the fields grid — and therefore destroy
focus — on a keystroke. `setSection` is documented as the only place that rebuilds fields, and
this would quietly make typing a second one.

**A branch on `sec.id` inside `buildFields`.** Greppable, so it satisfies the letter of the
registry rule, but not its purpose. The registry's promise is that adding a section type touches
`SECTIONS` and nothing else; one `if (sec.id === "arbitrary")` in the fields builder ends that
permanently, because the next non-scalar input adds a second and now there are two tables to keep
in step again — the exact drift one registry exists to prevent.

**A free-text box parsed on every keystroke, with no kind at all.** Tempting because the agreed
fragment encoding (`pts=0,3;0.5,0;2,0;2.5,3`) already implies a string, and one textarea is less
code than a kind mechanism. Rejected as the *only* option rather than outright: a textarea is a
defensible first control for the points kind. The kind is what lets it be replaced later by a
table or a drag-editable figure without touching validation, the fragment, or the sweep.

## What a kind owns

Five things the generic layers currently do inline, on the assumption that the value is a float:

- **Control** — what `buildFields` appends, and what carries the aria label.
- **Read** — how `readInputs` turns the control into a value. Today, `parseFloat(input.value)`.
- **Validate** — well-formedness of the value itself, which is what `domain` does for a scalar.
  Cross-parameter rules stay in `math.validate` and are unaffected.
- **Serialize** — how `writeHash` renders it and how `init()` restores it.
- **Sweepable** — whether the parameter may carry a sweep radio at all. `sweep: false` already
  exists and already covers `H`; for a points list it is false for a second, structural reason.

## Consequences

- **`readInputs` stops returning a map of floats.** Every `isFinite(v[k])` in the generic layers
  is really asking "is this scalar filled in", and each becomes a kind question: the "Fill in
  every field to see results." loop in `validate`, the `isFinite` filter in `writeHash`, and
  `stepFloor`/`stepCeiling`, which are scalar-only by definition and should say so.
- **`sweepPoint` already copies generically and needs no change.** It copies every key of `v`
  rather than naming a fixed set — a fix made for exactly this class of reason — so it carries a
  points array through untouched. It is the one place that is already right.
- **The fragment survives as agreed.** `readHash` splits on `&` and then on the first `=`, so
  `pts=0,3;0.5,0;2,0;2.5,3` round-trips without escaping. `writeHash` is the half that needs the
  kind, since its `isFinite` guard would otherwise drop the points silently — a shared link that
  opens on the right section type with no geometry in it.
- **`init()` is the path that decides whether a shared arbitrary section opens at all.** It
  applies hash values through `KEYS.forEach` behind the same `isFinite` guard, so restoring a
  points list is a kind-aware step there too.
- **The three parameter domains are untouched.** `positive`, `nonNegative` and `positiveAsPoint`
  remain what a *scalar* parameter declares. The near-sibling relationship between `validate` and
  `validateRange` is unaffected, because a points list is never a sweep range endpoint.
- The `view` half gains one obligation: `invertLabel(v)` returns `"b = 2 m"` or `"D = 2 m"` today,
  and an arbitrary section has no single defining dimension to name. What replaces it — a station
  range, a point count — is a `view` decision and must not leak into `math`.
