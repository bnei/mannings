# Sensitivity sweep, not an optimizer

The sweep feature holds every input fixed but one, evaluates the existing solver across a
range of that one, and plots the result. It was originally conceived as an "optimization
tool", and it is deliberately not one: there is no objective function, no inverse solve, and
no reported threshold crossing. It reports a response, not a recommendation.

Three features were on the table and they are genuinely different:

- **Sweep** — "how does normal depth respond to `n` between 0.02 and 0.05?" Answer: a curve.
- **Inverse solve** — "what bottom width gives exactly 1.2 m of depth at `Q` = 5?" Answer: a
  number, found by root-finding on the swept parameter.
- **Optimization** — "what `b`/`z` minimises excavation subject to `Q` ≥ 5?" Answer: a
  constrained minimum, the classic most-efficient-section problem.

The sweep is the substrate for all three — the curve *is* the function the other two would
search — so building it first costs nothing and forecloses nothing. Only the sweep ships.

The explicit no that matters most: the chart shades the region where the channel overtops but
does **not** report the parameter value at the crossing, even though that number is what an
engineer would write in a report. Reporting it means a second root-find and three new edge
cases (no crossing in range, multiple crossings, crossing at an endpoint), and it re-opens
inverse solve by the back door — the next request is inevitably an arbitrary target rather
than the fixed `H`. If that number is wanted later, add it deliberately as an inverse solve
with its own error states, not as a quiet extra line in the results panel.

## Consequences

Anyone asking why a tool with a parametric chart cannot answer "what `b` do I need?" should
read this first. The answer is that it can, by eye, and that making it answer numerically is
a scope decision rather than an oversight.
