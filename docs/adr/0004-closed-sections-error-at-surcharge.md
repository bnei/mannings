# Closed sections error at surcharge rather than reporting a result

For an open section, water above the top of bank is a *result with a warning*: the section
overtops, the banks are treated as continuing upward, and the numbers still describe uniform
flow down a channel. For a closed section, water reaching the crown is an **error**: no results,
an explanation instead.

The two look symmetrical and are not. Overtopping keeps a free surface, so Manning still applies
to the section that is drawn — the caveat is about whether the channel matches reality above the
bank, which is the user's judgement to make. Surcharge removes the free surface. Flow becomes
pressurized and is driven by the hydraulic grade line rather than by the bed slope, so `S₀` stops
being the energy slope and the equation being solved is no longer the equation the user asked
for. Continuing to report `Q` from Manning's equation there would produce a number that is wrong
for reasons nothing on the page mentions.

Extending the section upward, the way overtopping extends the banks, is not available either:
there is no defensible way to continue a circle past its crown.

## Where the line sits

At or above the crown. `y = D` is already surcharged — the free surface has zero width, the
hydraulic depth diverges and the Froude number is meaningless — so the error path takes `y ≥ D`,
not `y > D`.

Below that, the approach to the crown is degraded rather than invalid, and is reported as a
named condition: **nearly closed**, above `y/D` = 0.9, sibling to near critical. The section
properties are still correct there; it is the free-surface quantities — hydraulic depth, Froude
number, flow regime — that stop being meaningful as the surface narrows.

## Consequences

- Pressurized flow is out of scope for this tool, and this is the ADR that says so. Adding it
  later means a second equation (Darcy–Weisbach or Hazen–Williams), an upstream and a downstream
  head, and a results panel that reports something other than normal depth. That is a different
  tool, not a checkbox.
- Depth mode reaches the same wall from the other side: a discharge above the section's
  **maximum discharge** has no normal depth at all, and the error says that rather than blaming
  the geometry.
- **Capacity** stays "the discharge at the containing dimension" — at the crown for a closed
  section — even though the section conveys more just below it. The maximum discharge is
  reported as a caveat on capacity, never as the capacity: a designer sizing a culvert to its
  peak conveyance is designing to a depth the flow cannot be held at.
- Depth *above* the crown is never drawn, so the cross-section figure has no surcharged state to
  render.
