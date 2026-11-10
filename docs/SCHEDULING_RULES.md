# ShiftOpt — Scheduling Rules

**Version:** 0.1
**Status:** hard rules accepted; new defaults marked *proposed* pending supervisor review.
**Companion documents:** `docs/REQUIREMENTS.md` (product contract), `docs/OBJECTIVES.md` (scoring), `docs/ALTERNATIVES.md` (diversity).

This document defines what makes a schedule **valid** (hard constraints) and what makes it **good** (soft goals). It is the single source of truth for scheduling parameters and defaults.

## 1. Terminology

| Term | Meaning |
|---|---|
| Slot | One cell on the scheduling grid; default 15 minutes. |
| Operating slot | A slot inside the workplace's open hours for that day. |
| Shift | A maximal contiguous run of assigned slots for one employee on one day. |
| Assignment | An `(employee, day, slot)` triple where the employee works. |
| Assignment set `A` | The set of all assignments in a schedule. |
| Staff count `staff(s)` | Number of employees assigned to slot `s`. |
| Week | Monday–Sunday, recurring, wall-clock local time, no calendar dates. |

## 2. Hard constraints

A schedule is returned only if it satisfies **every** rule below. The independent validator re-checks all of them.

### H1 — Availability

An employee may be assigned only to slots that are free after their busy periods, the configured buffer, and clipping to operating hours.

- Busy periods and buffers are defined in `docs/INPUT_SCHEMA.md` §4.4–4.5.
- Busy time outside operating hours is ignored for scheduling.
- If buffering removes the whole day, the employee cannot work that day.

### H2 — Minimum staffing

For every operating slot `s`:

```
staff(s) >= minimum(s)
```

- `minimum(s)` comes from the staffing windows; default is 2.
- A supervisor may explicitly set `minimum = 1` for selected low-demand windows. This is a configuration change, not a solver relaxation.
- `minimum(s)` must not exceed the number of employees available in `s`; otherwise the instance is infeasible (see §5).

### H3 — Weekly hour cap

For every employee `e`:

```
assigned_minutes(e) <= max_weekly_hours * 60
```

- Default: 20 hours, workplace-wide.
- Per-employee overrides are a later feature.
- The cap applies to assigned shift time. Breaks are not modeled in the MVP.

### H4 — At most one shift per employee per day

An employee works at most one contiguous block per day. Split shifts are not allowed in the MVP.

### H5 — Contiguous shift

Assignments for one employee on one day form a single contiguous block:

```
if x[e,d,s] = 1 and x[e,d,s+2] = 1 then x[e,d,s+1] = 1
```

### H6 — Minimum shift length

Every non-empty shift has duration at least `min_shift_minutes`.

- Default: **120 minutes** (proposed).
- A free window shorter than this cannot host a shift; it is simply left unassigned.
- This rule can create coverage gaps that the validator must catch (see §5, F4).

### H7 — Maximum shift length

Every shift has duration at most `max_shift_minutes`.

- Default: **480 minutes / 8 hours** (proposed).
- With the default 11-hour operating day, two-person coverage, and an 8-hour cap, each day needs at least four shifts (for example, two `08:00–11:00` and two `11:00–19:00`). This is expected, not an error.

### H8 — Maximum consecutive working days

No employee works more than `max_consecutive_days` consecutive calendar days.

- Default: **5** (proposed).
- The check is circular inside the recurring week: if Saturday and Sunday are open, Sunday→Monday counts as consecutive.
- The MVP has no history from the previous week, so cross-week carry-over is out of scope (documented limitation).

### H9 — Minimum rest between shifts

For shifts on consecutive days `d` and `d+1`:

```
rest_minutes = (1440 - end(d)) + start(d+1) >= min_rest_minutes
```

- Default: **660 minutes / 11 hours** (proposed).
- With operating hours `08:00–19:00`, the maximum possible shift ends at 19:00 and the next starts at 08:00, giving 13 hours of rest, so the rule is non-binding under defaults. It becomes relevant if operating hours extend.
- The Sunday→Monday boundary is included when both days are open.

### H10 — Operating hours

- No assignment outside `open`–`close`.
- No assignment on a closed day.
- Opening and closing times must align with the grid.

### H11 — Time-grid alignment

All shift starts and ends align to the configured resolution (default 15 minutes). Imported times are snapped conservatively as described in `docs/INPUT_SCHEMA.md` §4.5.

### H12 — Single assignment per slot

Each `(employee, day, slot)` is binary: an employee is either assigned or not, never counted twice.

## 3. Soft goals

Soft goals never invalidate a schedule. They are scored in `docs/OBJECTIVES.md`.

| ID | Goal | Default preference |
|---|---|---|
| S1 | Target staffing | Prefer `staff(s) = target(s)` above the minimum. |
| S2 | Desired weekly hours | Prefer `assigned_minutes(e) ≈ desired_weekly_minutes(e)`. |
| S3 | Fairness | Prefer a small spread between the most- and least-assigned employee. |
| S4 | Shift duration | Prefer shifts at least 180 minutes; avoid unnecessary extra shifts. |
| S5 | Preferred shifts | Prefer assignments inside preferred windows (later feature). |
| S6 | Handover | Prefer at least 15 minutes of overlap when one employee hands a slot to another. |
| S7 | Mentorship | Prefer at least one mentor on duty whenever a new hire is on duty. |
| S8 | Skills | Prefer required skills per window (later feature). |

## 4. Defaults

Workplace settings are editable in the UI. Values marked **proposed** are pending supervisor confirmation; they are documented here so the model and tests can start.

| Parameter | Default | Status | Notes |
|---|---|---|---|
| Operating hours | Mon–Thu 08:00–19:00; Fri 08:00–17:00; Sat–Sun closed | confirmed | TSS preset. |
| Target staffing | 2 on duty | confirmed | Soft. |
| Minimum staffing | 2 on duty; 1 only in explicitly configured windows | confirmed | Hard. |
| Maximum weekly hours | 20 hours per employee | confirmed | Hard. |
| Scheduling resolution | 15 minutes | confirmed | Hard grid. |
| Buffer around busy periods | 15 minutes before and after | confirmed | Applied before clipping. |
| Preferred minimum shift | 180 minutes | confirmed | Soft. |
| Hard minimum shift | 120 minutes | **proposed** | Hard. |
| Maximum shift | 480 minutes | **proposed** | Hard. |
| Maximum consecutive days | 5 | **proposed** | Hard; circular within the week. |
| Minimum rest between shifts | 660 minutes (11 h) | **proposed** | Hard; non-binding under default hours. |
| One shift per day | yes | **proposed** | Hard; no split shifts. |
| Breaks modeled | no | **proposed** | Shifts are contiguous working blocks. |
| Preferred handover overlap | 15 minutes | confirmed | Soft. |
| Alternatives requested | 5 | confirmed | See `docs/ALTERNATIVES.md`. |
| Solver time limit | 30 seconds | confirmed | Initial default. |

If `max_weekly_hours < min_shift_minutes / 60`, the configuration is invalid: nobody could ever work.

## 5. Feasibility pre-checks and diagnostics

These checks run **before** the solver and again in the independent validator. They turn "no schedule found" into an actionable explanation.

### F1 — Configuration sanity

- `minimum <= target` in every window.
- `min_shift <= max_shift`.
- `max_shift <= longest operating window`.
- `max_weekly_hours * 60 >= min_shift`.
- Resolution divides buffer, shift lengths, and operating boundaries.

Failure code: `E-FEAS-001`.

### F2 — Per-slot availability

For every operating slot:

```
available_employees(s) >= minimum(s)
```

Failure code: `E-FEAS-002`, reported as merged time ranges and the missing headcount per range.

### F3 — Aggregate capacity

```
D = sum over operating slots of minimum(s) * resolution      # demand in staff-minutes
C = sum over employees of min(max_weekly_hours * 60, available_minutes(e))
```

If `C < D`, the instance is infeasible. Report:

- deficit in staff-hours;
- a lower bound on additional employees: `ceil((D - C) / (max_weekly_hours * 60))`, assuming full availability.

Failure code: `E-FEAS-003`.

### F4 — Shift-aware per-day coverability

Aggregate capacity can pass while shift rules still make a day impossible. For each open day, check whether the operating window can be covered at depth `minimum` using shifts between `min_shift` and `max_shift`.

A simple lower bound: at least `ceil(minimum * window_minutes / max_shift)` shifts are needed, and each contributes at least `min_shift`. A DP/greedy check in the validator confirms the exact result.

Failure code: `E-FEAS-004`. Simple examples: an operating window shorter than the minimum shift cannot host any shift; a window longer than the maximum shift must be tiled with overlapping shifts, and the check verifies that enough employees and hours exist to do so. The validator reports the uncovered remainder.

### F5 — Per-employee-day fit

For each employee and day, the longest free contiguous run must be at least `min_shift_minutes`; otherwise that employee cannot work that day.

This is a warning, not an error: `W-FEAS-005`. If it reduces availability below the minimum in some slot, F2 already fails.

### F6 — Desired hours vs demand

- If `sum(desired_hours) < demand_hours`, desired targets cannot all be met; warn `W-FEAS-006`. This is expected in small teams.
- If `sum(desired_hours) > capacity_hours`, warn `W-FEAS-007`; the solver will leave some desired hours unmet.

Neither warning makes the instance infeasible.

### F7 — Target vs availability

If `target(s) > available_employees(s)`, the target cannot be met even though the minimum might be. Warn `W-FEAS-008`; the schedule is still valid.

### Diagnostic output

Every infeasible instance returns:

1. the first binding check (`E-FEAS-*`),
2. the affected days, time ranges, and employees,
3. a quantitative deficit where possible,
4. at least one concrete suggested action, ordered by impact:
   - add employees (with the lower-bound count),
   - raise the weekly cap,
   - lower the minimum staffing in specific windows,
   - reduce the buffer,
   - allow longer shifts,
   - explicitly allow split shifts or more consecutive days.

## 6. Relaxation policy

The system **never** relaxes a hard rule silently. Relaxation is an explicit supervisor action, previewed and recorded in the output.

Never relaxed without an explicit override:

- H1 availability,
- H10 operating hours,
- H11 time grid,
- H12 single assignment,
- H4 one shift per day (split shifts require an explicit override).

Default relaxation order, from least to most disruptive:

1. maximum consecutive days,
2. maximum shift length,
3. minimum shift length,
4. weekly hour cap,
5. minimum staffing in selected windows.

Each step must show what changes, which employees are affected, and which rule was relaxed. A schedule produced after relaxation is labelled as such and cannot be compared as if it were fully compliant.

## 7. Assumptions and limitations

- No unpaid breaks are modeled; shifts are contiguous working blocks. Break policy is handled by the supervisor.
- No split shifts in the MVP.
- No overnight shifts; operating hours stay within one calendar day.
- Wall-clock local time only; no time zones, daylight saving, dates, or holidays.
- Weekly recurring pattern only.
- The weekly hour cap is workplace-wide in the MVP.
- No cross-week carry-over of consecutive days or rest; the check is circular within the seven-day week.
- Single location only.
- Desired weekly hours are soft targets; minimum staffing is hard.
- Tags are informational in the MVP; mentorship is a soft pairing goal, not a hard skill requirement.
