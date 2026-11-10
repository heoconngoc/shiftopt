# ShiftOpt — Alternative Schedules and Diversity

**Version:** 0.1
**Status:** metric and selection method accepted; thresholds are **provisional**.
**Companion documents:** `docs/OBJECTIVES.md` (scoring), `docs/SCHEDULING_RULES.md` (validity).

## 1. Purpose

The product promise is not "one optimal schedule". It is "several useful schedules that differ in ways a supervisor can understand and choose between". Reordering the same assignments, or changing one 15-minute slot, does not count.

This document defines:

- how similarity between two schedules is measured;
- how many alternatives to generate and how they are selected;
- when an alternative is too similar or too poor to show;
- how the system behaves when it cannot find enough diverse options.

## 2. Representations

For a schedule `P`:

- `A(P)` = set of assignments `(employee, day, slot)`.
- `B(P)` = multiset of shift blocks `(day, start, end)`, ignoring employee identity.

`A` captures **who works when**. `B` captures the **shape of the day** (when shifts start and end). Both are needed:

- two schedules that swap employees but keep identical shift boundaries have `d_assign > 0` and `d_struct = 0`;
- two schedules that keep employees but move shift boundaries have `d_assign > 0` and `d_struct > 0`;
- two schedules with different employees and different boundaries differ on both.

## 3. Distance metrics

### 3.1 Assignment distance

```
d_assign(P, Q) = 1 - 2 * |A(P) ∩ A(Q)| / (|A(P)| + |A(Q)|)
```

- `0` = identical assignment sets.
- `1` = no shared assignment.
- If both sets are empty, define `d_assign = 0`.
- This is a Dice distance; it is symmetric and bounded in `[0, 1]`.

With a typical instance of 424 staff-slots (106 staff-hours on a 15-minute grid), a threshold of `0.20` means roughly 85 changed slot-assignments, or about 21 staff-hours.

### 3.2 Shift-structure distance

```
d_struct(P, Q) = 1 - 2 * |B(P) ∩ B(Q)| / (|B(P)| + |B(Q)|)
```

`B` is a multiset; intersection counts the minimum multiplicity of each `(day, start, end)` block.

- `0` = identical shift boundaries.
- `1` = no shared shift boundary.
- If both multisets are empty, define `d_struct = 0`.

### 3.3 Pairwise requirement

An alternative is diverse enough only if, against **every already selected alternative**:

```
d_assign >= theta_assign
and
d_struct >= theta_struct
```

Using the minimum against the whole selected set avoids a chain where A is close to B, B is close to C, but A and C are presented as different.

### 3.4 Default thresholds

| Threshold | Default | Status |
|---|---|---|
| `theta_assign` | 0.20 | proposed |
| `theta_struct` | 0.10 | proposed |
| Near-duplicate guard | `d_assign < 0.02` always rejected | proposed |

Thresholds are advanced settings. They must be validated with real supervisor feedback; a supervisor may consider 20% too similar or too different depending on team size.

## 4. Quality gate

Diversity is worthless if the alternatives are bad. A candidate is eligible only if:

1. it passes the independent validator (all hard rules);
2. its under-target coverage is no worse than the best schedule: `U <= U_best`;
3. its penalty is within a relative slack of the best:

```
P <= P_best + 0.10 * max(P_best, 1)
```

The `max(P_best, 1)` term prevents a zero-penalty best schedule from making the gate impossible.

Default slack: **10%**. The slack is relaxed only in the fallback path (§7).

## 5. Candidate generation

The solver must produce more than one optimum. The planned approach:

1. Solve for the best schedule by `(U, P)`.
2. Generate additional candidates with a solution pool, no-good cuts, or repeated solves that exclude already-found assignment sets.
3. Perturb the objective to explore different trade-off regions:
   - random multipliers of `±20%` on the secondary weights;
   - alternate emphasis on fairness, desired hours, and handover;
   - different random seeds.
4. Cap the candidate pool (default 200) or split the time budget, for example 60% for the best schedule and 40% for diversity.
5. Run the independent validator on every candidate; discard invalid ones.

The candidate pool is internal. The supervisor sees only selected alternatives.

## 6. Selection algorithm

Use farthest-first (max–min) selection:

1. Start with the best schedule by `(U, P)`.
2. Among eligible candidates, compute the minimum `d_assign` and `d_struct` against the selected set.
3. Choose the candidate that maximises the minimum `d_assign`, subject to both thresholds.
4. Tie-break by larger minimum `d_struct`, then by lower `P`.
5. Repeat until `K` alternatives are selected or no candidate passes.

This is deterministic given a fixed candidate order and seed.

Why not simply take the top `K` by penalty? Top-`K` optima are usually near-duplicates. Why not cluster first? Clustering needs a distance threshold anyway; farthest-first is simpler, explainable, and directly optimises the property the product promises.

## 7. Fallback when fewer than `K` alternatives exist

Never pad the list with near-duplicates. Relax in this order:

1. Widen the quality gate to 20% and retry.
2. Lower the thresholds to `theta_assign = 0.15`, `theta_struct = 0.075` and retry.
3. Return fewer than `K` alternatives with an explanation.

The explanation must say which limit was hit, for example:

- "Only 3 alternatives meet the quality and diversity thresholds; the remaining candidates differ by less than 15% of assignments."
- "The team is too small for more diverse schedules without exceeding the weekly cap."

## 8. Output per alternative

Each alternative carries:

| Field | Content |
|---|---|
| `id` | Stable label (`A`, `B`, `C`, …). |
| `metrics` | `U`, `O`, `H`, `F`, `Sq`, `R`, `X`, `M` from `docs/OBJECTIVES.md`. |
| `deltas` | Difference from the best alternative for each metric. |
| `distances` | Minimum `d_assign` and `d_struct` against the other selected alternatives. |
| `tradeoff` | One or two generated sentences naming the main gains and losses. |
| `relaxed` | Which hard rules, if any, were explicitly relaxed, and by whom. |
| `seed` | Seed used to generate it, for reproducibility. |

## 9. Determinism

- Fixed seed and documented solver version.
- Stable candidate ordering; ties broken by a hash of the assignment set.
- The same input, settings, and seed must produce the same alternatives in the same order.
- If a time limit truncates the candidate pool, the result is still deterministic for that time limit; changing the limit may change the set, and the UI must show the limit used.

## 10. Benchmark targets

Before release, measure on synthetic and anonymised instances:

| Measure | Target |
|---|---|
| Valid schedules | 100% pass the independent validator. |
| Feasible instances with enough diversity | At least 90% return `K - 1` or more alternatives above thresholds. |
| Mean pairwise `d_assign` | At least 0.25. |
| Mean pairwise `d_struct` | At least 0.12. |
| Penalty gap of selected alternatives | At most 10% above the best. |
| Runtime | Within the configured solver limit. |
| Duplicate alternatives | Zero. |

Store raw results in `experiments/` and summarise them here.

## 11. Anti-patterns

- Enumerating every feasible schedule: combinatorially explosive and not useful.
- Presenting the same assignment set with a different employee order.
- Counting a 15-minute jitter as a new alternative.
- Hiding a dominated alternative without saying why.
- Returning fewer alternatives silently.
- Calling an alternative "best" without naming the metric on which it is best.

## 12. Open questions

- Should diversity be measured for the whole week or per day as well?
- Is an employee-swap-only alternative useful to the supervisor? `d_assign` says yes; `d_struct` says the day shape is unchanged. The current rule requires both, which rejects pure swaps.
- Should the UI offer "more like this" and "show me something more different" controls?
- Are the 20% / 10% thresholds right for a 5-person team versus a 20-person team?
- Should the number of alternatives scale with team size or stay fixed at 5?
