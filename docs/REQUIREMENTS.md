# ShiftOpt — Requirements

## 1. Overview

ShiftOpt is a privacy-first scheduling tool that helps supervisors generate and compare work schedules.

**Design philosophy: Simple by default, powerful when needed.**

Users can generate schedules with minimal input, while advanced settings allow more control over scheduling constraints and optimization priorities.

The supervisor always makes the final decision.

## 2. Core Workflow

1. **Import** employee availability and preferences.
2. **Configure** operating hours and staffing needs.
3. **Generate** multiple feasible schedule options.
4. **Compare** schedules based on coverage, working hours, preferences, and trade-offs.
5. **Export** the preferred schedule.

No account or permanent data storage is required.

## 3. Basic Requirements

Everything needed for a simple schedule should be available on the main page.

### Employees

Required:
- Name

Optional:
- Recurring busy times (availability). A blank busy-time cell means no known conflict during operating hours.
- Desired weekly hours
- Preferred shifts
- Tags (e.g., new hire, mentor, skills)

Employee IDs are generated internally when needed. Users do not need to provide them.

### Workplace

- Configurable operating days and hours
- Desired number of employees on duty
- Minimum acceptable staffing level

### Scheduling

- Respect employee availability.
- Respect configured maximum working hours.
- Prefer meeting staffing targets.
- Generate multiple high-quality schedule alternatives.

### Mandatory scheduling rules

A returned schedule is valid only if it satisfies every rule in `docs/SCHEDULING_RULES.md`:

- availability after busy periods and buffers;
- minimum staffing in every operating slot;
- weekly hour cap;
- at most one contiguous shift per employee per day;
- minimum and maximum shift length;
- maximum consecutive working days and minimum rest between shifts;
- operating hours and time-grid alignment.

The system must never return a schedule that violates a hard rule.

### Quality goals

When several valid schedules exist, prefer those that:

- meet target staffing above the minimum;
- match desired weekly hours;
- distribute hours fairly;
- offer reasonable shift lengths and handovers;
- respect preferred shifts and mentorship needs.

Goal weights and tie-breaking are defined in `docs/OBJECTIVES.md`.

### Infeasibility

If no valid schedule exists, the system must explain why, identify the binding constraint, and suggest concrete changes. Relaxing a hard rule requires explicit supervisor action; the system must never relax silently.

## 4. Advanced Settings

Optional settings may include:

- Different staffing requirements by time of day
- Maximum weekly hours per employee
- Minimum and maximum shift length
- Maximum consecutive working days and minimum rest between shifts
- Preferred shift duration
- Travel or preparation buffers
- Handover overlap and coordination
- Mentorship and skill-based scheduling
- Optimization priorities and goal weights
- Number and diversity of schedule alternatives
- Solver time limit
- Random seed for reproducible alternatives
- Infeasibility relaxation policy

Advanced settings must have sensible defaults and should not be required for basic use.

## 5. Schedule Comparison

Each generated schedule should include:

- Weekly calendar view
- Assigned hours per employee
- Coverage summary
- Satisfaction of employee preferences
- Handover and mentorship information, when applicable
- Strengths and trade-offs compared with other options

Schedules should be meaningfully different, not merely reordered versions of the same assignments. Diversity is measured with the thresholds in `docs/ALTERNATIVES.md`. Each alternative must show its trade-offs against the others, not only its own score.

## 6. Privacy

- No account required.
- No permanent storage of imported employee data.
- No automatic saving to browser storage.
- Prefer local or browser-based processing.
- Clearly disclose if data must be sent to a server.
- Export only when requested by the user.
- Parse and validate imported data in memory; do not write it to disk, logs, or browser storage.
- If telemetry is ever added, record only aggregate, non-identifying diagnostics.

## 7. Out of Scope

- Payroll and timesheets
- Absence and shift-swap management
- HR integrations
- Multi-location scheduling

## 8. Success Criteria

ShiftOpt should:

- Produce valid schedules that respect mandatory constraints.
- Return no schedule that fails the independent validator.
- Reduce manual scheduling effort.
- Offer useful alternatives rather than a single unexplained result.
- On benchmark instances with a feasible K-alternative set, return at least K-1 alternatives above the diversity thresholds.
- Explain every infeasible instance with the binding constraint and at least one concrete suggestion.
- Make scheduling trade-offs understandable.
- Produce identical alternatives for identical input, settings, and seed.
- Remain easy to use for simple cases.
- Support more complex scheduling needs without overwhelming basic users.

## 9. Assumptions and Limitations

- No unpaid breaks are modeled; shifts are contiguous working blocks.
- No split shifts in the MVP.
- No overnight shifts; operating hours stay within one calendar day.
- Wall-clock local time only; no time zones, daylight saving, dates, or holidays.
- Weekly recurring pattern only.
- The weekly hour cap is workplace-wide in the MVP.
- No cross-week carry-over of consecutive days or rest; the check is circular within the seven-day week.
- Single location only.
- Desired weekly hours are soft targets; minimum staffing is hard.
- Tags are informational in the MVP; mentorship is a soft pairing goal.

## 10. Related Documents

- `docs/INPUT_SCHEMA.md` — CSV format, validation, and normalized input.
- `docs/SCHEDULING_RULES.md` — hard constraints, defaults, feasibility checks, relaxation.
- `docs/OBJECTIVES.md` — metrics, objective order, weights, trade-off display.
- `docs/ALTERNATIVES.md` — diversity metrics, selection, benchmark targets.
- `docs/DECISIONS.md` — decision log.