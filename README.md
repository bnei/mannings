# mannings

Manning's equation solver for open channels and closed conduits. Two section types ship —
**symmetric trapezoidal** and **circular** — picked from the dropdown in the title. Solves
either direction, normal depth from a discharge or discharge from a depth, and reports the
core hydraulics plus Froude number and flow regime.

It also draws a **sensitivity sweep**: hold every input fixed but one, and see the response
as a curve. The swept parameter is picked with the radio beside its field, over an explicit
range. The response is whichever quantity the current mode solves for, so sweeping the mode's
own fixed input gives the rating curve. Shading marks where the section would overtop or
surcharge, and a dot marks the values currently in the fields.

The sweep reports a response, not a recommendation — there is no inverse solve and no
optimizer. See [docs/adr/0001](./docs/adr/0001-sensitivity-sweep-not-an-optimizer.md).

SI units only (`k = 1.0`), so there is no unit conversion anywhere in the code.

## Running it

Open `index.html`. That's it — no build step, no dependencies, no server. It works from
`file://` and offline.

## Deploying

Live on Cloudflare Pages at [mannings.briannei.com](https://mannings.briannei.com), connected
to this repo — a push to `main` redeploys. No build command, output directory is the repo
root. Every request is a static asset, so there are no Functions invocations and nothing
metered.

## Method

Manning's equation and the quantities derived from it are the same for every section type:

```
R  = A / P                      hydraulic radius
Dh = A / T                      hydraulic depth
Q  = (1/n)·A·R^(2/3)·sqrt(S₀)   Manning, SI
V  = Q / A
Fr = V / sqrt(g·Dh)             g = 9.80665 m/s²
```

Only the section properties differ. Trapezoidal, with bottom width `b`, side slope `z` and
channel height `H`:

```
A = y(b + z·y)                  flow area
P = b + 2y·sqrt(1 + z²)         wetted perimeter
T = b + 2z·y                    top width
```

`z` is the side slope as horizontal run per unit rise (H:V). `b = 0` gives a triangular
section and `z = 0` a rectangular one; both are valid.

Circular, of diameter `D`, where `θ` is the angle subtended at the centre by the wetted
boundary:

```
θ = 2·acos(1 − 2y/D)
A = r²(θ − sin θ) / 2           r = D/2
P = r·θ
T = 2r·sin(θ/2)
```

These are analytic, never measured off the drawn outline. The outline exists to be drawn; a
polygon's perimeter under-reports a curved boundary, and `P` is two thirds of `R`.

Normal depth is found by **enumerating every root**: `Q(y) − Q` is sampled on a grid over the
depth range, every sign change is bisected, and the shallowest root is reported with the
existence of any others stated. There is no monotonicity assumption — a circular section's
conveyance peaks at `y/D` ≈ 0.938 and falls to the crown, so a band of discharge about 7% wide
has two normal depths. See
[docs/adr/0003](./docs/adr/0003-shallowest-root-conveyance-not-monotone.md). Bisection rather
than Newton, as before: a sign change brackets a root, there is no derivative to get wrong and
the iteration cannot diverge.

## Assumptions and limits

Uniform, steady flow in a prismatic channel. Results are flagged as unreliable within a few
percent of `Fr = 1`, and flagged when the computed depth exceeds the channel height of an open
section.

A closed section reaching its crown is an **error, not a warning**: flow becomes pressurized
and is no longer governed by a free surface, which is outside what this tool solves. Its
**capacity** is the discharge at the crown, and the greater discharge the section conveys just
below the crown is reported as a caveat on that rather than as the capacity. See
[docs/adr/0004](./docs/adr/0004-closed-sections-error-at-surcharge.md).

Backwater effects, composite roughness, critical-depth calculations and arbitrary or compound
sections are out of scope.

Inputs are mirrored into the URL fragment, so any configuration is bookmarkable — including
the section type, the swept parameter and its range, so a shared link reproduces the same
figure. A link with no section type is trapezoidal, so links predating section types still
open. The fragment is never transmitted to the server.

Sweep axes are linear, never logarithmic: a log axis cannot represent zero, and `b = 0` and
`z = 0` are valid sections that must stay sweepable. The response axis always starts at zero.
