# ShiftOpt — Objectives and Trade-offs

**Version:** 0.1
**Status:** structure accepted; default weights are **provisional** and must be validated with the supervisor.
**Companion documents:** `docs/SCHEDULING_RULES.md` (validity), `docs/ALTERNATIVES.md` (diversity), `docs/DECISIONS.md` (ADR log).

## 1. Principles

1. **Validity first.** A schedule that violates any hard rule is never scored or shown as an option.
2. **Coverage is primary.** Among valid schedules, fewer uncovered target slots always wins. The supervisor should never trade coverage for fairness without seeing it explicitly.
3. **No opaque single score.** The UI shows the metric vector and the trade-offs. A composite penalty is used internally for sorting, not presented as "the" quality of a schedule.
4. **Deterministic.** Same input, settings, and seed produce the same alternatives.
5. **Explainable.** Every alternative states what it gains and what it gives up compared with the best option.

## 2. Metrics

All metrics are computed from the assignment matrix `x[e,d,s]`, where `e` is an employee, `d` a day, and `s` a slot.

| Symbol | Metric | Unit | Direction |
|---|---|---|---|
| `U` | Under-target coverage: `sum(max(0, target(s) - staff(s)) * window_weight(s)) * resolution / 60` | staff-hours | lower is better |
| `O` | Over-target coverage: `sum(max(0, staff(s) - target(s)) * window_weight(s)) * resolution / 60` | staff-hours | lower is better |
| `H` | Desired-hours deviation: `sum(abs(assigned_hours(e) - desired_hours(e)))` for employees with a target | hours | lower is better |
| `F` | Fairness spread: `max(assigned_hours) - min(assigned_hours)` over employees with at least one assignable slot | hours | lower is better |
| `Sq` | Short-shift hours: `sum(max(0, preferred_min_shift - shift_duration))` | hours | lower is better |
| `R` | Off-preference hours: assigned hours outside preferred windows (0 until preferred shifts exist) | hours | lower is better |
| `X` | Handover gaps: number of shift transitions per day with overlap below the preferred handover | count | lower is better |
| `M` | Mentorship gaps: staff-hours where a new hire is on duty and no mentor is on duty | staff-hours | lower is better |

Notes:

- `staff(s)` is the number of employees assigned to slot `s`.
- `window_weight(s)` defaults to 1; critical windows may use 2 or 3. It is configured with the staffing windows, not learned.
- `H` ignores employees with no desired target (blank or 0). Fairness still covers them.
- `F` includes every employee who has at least one assignable slot in the week, so a fully unavailable employee does not distort the spread.
- `X` counts a transition when one employee's shift ends and another's begins on the same day. Overlap reduces the gap; a 15-minute overlap removes the penalty.
- `M` is zero when the team has no `new-hire` tag or no `mentor` tag.

## 3. Objective order

The solver compares valid schedules lexicographically:

1. **Minimize `U`.** Coverage against the target is the primary soft goal.
2. **Minimize `P`.** Secondary penalty, defined below.
3. **Tie-break:** fewer total assigned staff-hours (avoid unnecessary overstaffing), then a stable deterministic hash of the schedule.

```
P = w_O * O
  + w_H * H
  + w_F * F
  + w_S * Sq
  + w_R * R
  + w_X * X
  + w_M * M
```

`U` is deliberately outside `P`. No finite weight reliably prevents a solver from sacrificing a whole coverage slot for a small fairness gain; the lexicographic split makes the priority explicit.

Minimum staffing is a **hard** constraint (see `docs/SCHEDULING_RULES.md` H2), so `U` only measures the gap between the configured minimum and the higher target.

## 4. Default weights

All weights are per unit of the metric above. They are provisional and must be checked for sensitivity before release.

| Weight | Default | Unit | Rationale |
|---|---|---|---|
| `w_O` | 1.0 | per staff-hour | Overstaffing wastes hours but is less harmful than under-target coverage. |
| `w_H` | 2.0 | per hour | Desired hours are a strong employee preference. |
| `w_F` | 5.0 | per hour of spread | Fairness is a core promise; a 4-hour spread costs 20 points. |
| `w_S` | 1.0 | per hour below the preferred minimum | Mild preference for reasonable shift lengths. |
| `w_R` | 1.0 | per off-preference hour | Reserved for preferred shifts. |
| `w_X` | 1.0 | per handover gap | Mild preference for continuity. |
| `w_M` | 2.0 | per unmentored new-hire staff-hour | Mentorship matters but is not a hard skill rule in the MVP. |

### Priority levels

Advanced settings expose per-goal priority as **low / normal / high**, mapping to multipliers `0.5 / 1.0 / 2.0` on the corresponding weight. Coverage `U` is not user-weightable; only the per-window weights change it.

## 5. Comparison and trade-off presentation

Each alternative is displayed with:

1. **Coverage:** target slots met, under-target staff-hours, over-target staff-hours.
2. **Hours:** assigned hours per employee, desired-hours deviation, fairness spread.
3. **Shift quality:** number of shifts, short-shift hours, handover gaps.
4. **Preferences and mentorship:** off-preference hours, unmentored new-hire hours.
5. **Deltas** against the best alternative for each metric.
6. **One or two sentences** describing the main trade-off.

### Pareto filtering

Alternative `B` is **dominated** by `A` when `A` is no worse on every metric and strictly better on at least one. Dominated alternatives are hidden unless they are needed to satisfy the diversity rules in `docs/ALTERNATIVES.md`. This prevents presenting an option that is worse in every way.

### Trade-off sentences

Sentences are generated from the largest metric deltas, for example:

- "Covers the target fully, but Alice works 4 hours more than Bob and there are 2 handover gaps."
- "Falls 3 slots (0.75 staff-hours) short of the target, but the hours spread is 3 hours smaller."
- "Meets desired hours for 10 of 12 employees; the two exceptions are the employees with the least availability."

The wording must name the metric and the affected employees or time ranges, never just say "better score".

### What the UI must not do

- Do not show only a composite score.
- Do not label an alternative "best" without saying best on which metric.
- Do not hide a trade-off because the composite penalty is close.
- Do not compare a relaxed schedule as if it were fully compliant.

## 6. Determinism and sensitivity

- The solver runs with a fixed random seed and a documented time limit.
- Identical input, settings, and seed must produce identical alternatives; if the solver cannot guarantee this, a deterministic post-selection and tie-break restore it.
- Before release, run a sensitivity check: vary each weight by ±50% and record whether the top-three ranking changes. Store results in `experiments/` and summarise them in this document.

## 7. Worked example

Two valid schedules with the same `U = 0`:

| Metric | A | B |
|---|---|---|
| Over-target `O` | 2 staff-hours | 6 staff-hours |
| Desired deviation `H` | 8 hours | 2 hours |
| Fairness spread `F` | 2 hours | 1 hour |
| Short-shift `Sq` | 1 hour | 3 hours |
| Handover gaps `X` | 1 | 3 |

```
P(A) = 1*2 + 2*8 + 5*2 + 1*1 + 1*1 = 2 + 16 + 10 + 1 + 1 = 30
P(B) = 1*6 + 2*2 + 5*1 + 1*3 + 1*3 = 6 +  4 +  5 + 3 + 3 = 21
```

`B` is preferred: it gives up 4 staff-hours of overstaffing and 2 handover gaps, but gains 6 hours of desired-hours accuracy and a smaller fairness spread. The UI sentence would be:

> "B matches desired hours better and is fairer by 1 hour, at the cost of 4 extra staff-hours and 2 more handover gaps."

## 8. Open questions

- Are the provisional weights aligned with how the supervisor actually chooses between schedules?
- Should fairness use standard deviation instead of max–min spread? Spread is easier to explain; standard deviation punishes a single outlier more.
- Should handover gaps count as a binary transition or as missing overlap minutes?
- Should coverage target weight differ by day (for example, busier Mondays)?
- Should employees with zero availability be excluded from fairness, or shown separately as a staffing risk?
