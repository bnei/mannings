# mannings

Manning's equation solver for symmetric trapezoidal channels. Solves either direction —
normal depth from a discharge, or discharge from a depth — and reports the core hydraulics
plus Froude number and flow regime.

SI units only (`k = 1.0`), so there is no unit conversion anywhere in the code.

## Running it

Open `index.html`. That's it — no build step, no dependencies, no server. It works from
`file://` and offline.

## Deploying

Static site on Cloudflare Pages. Every request is a static asset, so there are no Functions
invocations and nothing metered.

## Method

```
A  = y(b + z·y)                 flow area
P  = b + 2y·sqrt(1 + z²)        wetted perimeter
R  = A / P                      hydraulic radius
T  = b + 2z·y                   top width
D  = A / T                      hydraulic depth
Q  = (1/n)·A·R^(2/3)·sqrt(S₀)   Manning, SI
V  = Q / A
Fr = V / sqrt(g·D)              g = 9.80665 m/s²
```

`z` is the side slope as horizontal run per unit rise (H:V). `b = 0` gives a triangular
section and `z = 0` a rectangular one; both are valid.

Normal depth is found by bisection. `Q(y)` is strictly increasing in `y` for a trapezoid,
so a sign change brackets a unique root and the iteration cannot diverge — chosen over
Newton deliberately, since there is no derivative to get wrong and no failure mode to
explain to a reviewer.

## Assumptions and limits

Uniform, steady flow in a prismatic channel. Results are flagged as unreliable within a
few percent of `Fr = 1`, and flagged when the computed depth exceeds the channel height.
Backwater effects, non-uniform sections, composite roughness and critical-depth
calculations are out of scope.

Inputs are mirrored into the URL fragment, so any configuration is bookmarkable. The
fragment is never transmitted to the server.
