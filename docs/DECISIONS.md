# ShiftOpt — Decision Log

Architecture and product decisions that shape the solver, the input contract, or the public documentation. Each entry records the context, the decision, and the consequences so later changes are deliberate.

Status values: **accepted**, **proposed**, **superseded**.

---

## ADR-001 — One matrix CSV for employee import

**Date:** 2026-11-10
**Status:** accepted (supersedes the two-file long format in schema 0.1)

**Context.** The first schema used `employees.csv` plus a long-format `busy_times.csv`. Supervisors already work in spreadsheets, and a long format needs a second file, a join key, and more validation paths.

**Decision.** Use one file, one row per employee, one column per weekday, busy ranges inside cells (`09:00-10:15; 13:00-14:00`). Only `Name` is required. Internal IDs are generated after validation.

**Consequences.**

- One upload, one validation report, fewer user errors.
- Cells are longer and harder to edit by hand than a long format; the future calendar editor is the answer.
- Preferred shifts do not fit naturally; they stay out of the MVP CSV.
- Schema 0.1 is superseded.

---

## ADR-002 — Minimum staffing is hard, target staffing is soft

**Date:** 2026-11-10
**Status:** accepted

**Context.** Requirements mention both a "desired number of employees on duty" and a "minimum acceptable staffing level". Treating both as hard makes small teams infeasible; treating both as soft can return schedules that leave the workplace understaffed.

**Decision.** `minimum(s)` is a hard constraint. `target(s)` above the minimum is the primary soft goal. A supervisor may explicitly set `minimum = 1` in selected low-demand windows.

**Consequences.**

- Every returned schedule satisfies the minimum or is reported as infeasible.
- Coverage quality is measured as the gap to the target (`U`), not the gap to the minimum.
- Infeasibility messages must explain how to lower the minimum or add staff.

---

## ADR-003 — Snap imported times outward instead of rejecting them

**Date:** 2026-11-10
**Status:** accepted

**Context.** Schema 0.2 rejected intervals that were not aligned to the 15-minute grid. Supervisors type approximate times; rejection creates friction and repeated uploads.

**Decision.** Snap the start down and the end up to the nearest grid line, then emit `W-TIME-001`. Never snap inward.

**Consequences.**

- Import succeeds more often.
- The employee is never made *more* available than the file says, so the solver cannot over-book.
- The warning must show both the original and normalized interval.
- Exact times still matter for auditability; the original values stay in the diagnostic.

---

## ADR-004 — Real-schedule hard constraints and their defaults

**Date:** 2026-11-10
**Status:** proposed (defaults pending supervisor review)

**Context.** The original requirements covered availability, coverage, and the weekly cap, but not maximum shift length, consecutive days, rest, or split shifts. A model without them can return 11-hour shifts or seven-day runs.

**Decision.** Add hard rules H4–H9 in `docs/SCHEDULING_RULES.md`:

- one contiguous shift per employee per day;
- minimum shift 120 minutes;
- maximum shift 480 minutes;
- maximum 5 consecutive working days;
- minimum rest 660 minutes between shifts;
- no breaks modeled in the MVP.

**Consequences.**

- An 11-hour operating day with two-person coverage needs at least four shifts per day; short morning plus long day shifts become normal.
- Small teams become infeasible sooner; the pre-checks must explain why.
- Break handling stays a supervisor responsibility and must be stated in the UI.
- Defaults are marked *proposed* until the supervisor confirms them.

---

## ADR-005 — Coverage-first lexicographic objective with a weighted secondary penalty

**Date:** 2026-11-10
**Status:** proposed (weights provisional)

**Context.** A pure weighted sum lets the solver trade one uncovered slot for a large fairness or preference gain. A pure lexicographic order over seven goals is brittle and hard to explain. The supervisor must see trade-offs, not a single score.

**Decision.** Two-level objective:

1. minimise under-target coverage `U`;
2. minimise `P = w_O*O + w_H*H + w_F*F + w_S*Sq + w_R*R + w_X*X + w_M*M`;
3. tie-break by fewer assigned staff-hours, then a stable hash.

Expose per-goal priority levels (low/normal/high) and show the metric vector with deltas. Details in `docs/OBJECTIVES.md`.

**Consequences.**

- Coverage is never silently traded away.
- Fairness, desired hours, shift quality, and mentorship compete inside `P`; their weights need sensitivity testing.
- The UI must not present `P` as "the" score.
- Weights are provisional and must be validated with real choices.

---

## ADR-006 — Diversity as assignment distance plus shift-structure distance

**Date:** 2026-11-10
**Status:** proposed (thresholds provisional)

**Context.** "Meaningfully different" was not measurable. Top-`K` optima are usually near-duplicates, and employee swaps with identical shift boundaries may or may not count as different.

**Decision.**

- `d_assign` = Dice distance on `(employee, day, slot)` assignments.
- `d_struct` = Dice distance on `(day, start, end)` shift blocks, ignoring employee.
- An alternative must pass both thresholds against every selected alternative.
- Selection is farthest-first (max–min) with a quality gate.
- Defaults: `theta_assign = 0.20`, `theta_struct = 0.10`, quality slack 10%.

Details in `docs/ALTERNATIVES.md`.

**Consequences.**

- Pure employee swaps are rejected because the day shape is unchanged.
- The system can return fewer than `K` alternatives; it must explain why.
- Thresholds need supervisor feedback and benchmark data.
- The solver needs a candidate pool, not a single optimisation run.

---

## ADR-007 — Local AI notes stay out of the public repository

**Date:** 2026-11-10
**Status:** accepted

**Context.** `AI/CONTEXT.md`, `AI/PLAN.md`, and `AI/NEXT.md` are working notes for the agent and the user. They contain process details that do not belong in the product repository.

**Decision.** Keep `AI/` ignored via `.gitignore` and do not commit it. The public contract lives in `docs/`. `AI/CONTEXT.md` may reference the `AI/` files, but the paths must be correct.

**Consequences.**

- GitHub shows only product documents, code, and synthetic examples.
- The notes are not versioned; they can drift or be lost. Important decisions must be copied into `docs/DECISIONS.md` or the relevant contract document.
- New machines do not get the notes automatically.
