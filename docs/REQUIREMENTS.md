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
- Availability

Optional:
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

## 4. Advanced Settings

Optional settings may include:

- Different staffing requirements by time of day
- Maximum weekly hours per employee
- Preferred shift duration
- Travel or preparation buffers
- Handover overlap and coordination
- Mentorship and skill-based scheduling
- Optimization priorities
- Number and diversity of schedule alternatives
- Solver time limit

Advanced settings must have sensible defaults and should not be required for basic use.

## 5. Schedule Comparison

Each generated schedule should include:

- Weekly calendar view
- Assigned hours per employee
- Coverage summary
- Satisfaction of employee preferences
- Handover and mentorship information, when applicable
- Strengths and trade-offs compared with other options

Schedules should be meaningfully different, not merely reordered versions of the same assignments.

## 6. Privacy

- No account required.
- No permanent storage of imported employee data.
- No automatic saving to browser storage.
- Prefer local or browser-based processing.
- Clearly disclose if data must be sent to a server.
- Export only when requested by the user.

## 7. Out of Scope

- Payroll and timesheets
- Absence and shift-swap management
- HR integrations
- Multi-location scheduling

## 8. Success Criteria

ShiftOpt should:

- Produce valid schedules that respect mandatory constraints.
- Reduce manual scheduling effort.
- Offer useful alternatives rather than a single unexplained result.
- Make scheduling trade-offs understandable.
- Remain easy to use for simple cases.
- Support more complex scheduling needs without overwhelming basic users.