# Examples

Synthetic fixtures only. Never commit real employee data.

These files are for schema review, manual testing, and future parser tests. They are not production data.

## Files

| File | Purpose |
|---|---|
| `team_template.csv` | Small schema demo. **Not feasible** under the default workplace settings; use it to test infeasibility messaging. |
| `team_synthetic_12.csv` | Feasible synthetic team for normal generation and benchmarks. |
| `invalid/*.csv` | Files that must be rejected with the listed error codes. |
| `warnings/*.csv` | Files that must import successfully but emit the listed warnings. |

## Why `team_template.csv` is not feasible

Default demand (see `docs/SCHEDULING_RULES.md`):

- Mon–Thu: 11 open hours × 2 staff = 22 staff-hours/day × 4 = 88.
- Fri: 9 open hours × 2 staff = 18.
- Weekly demand = **106 staff-hours**.

`team_template.csv` has 3 employees and a 20-hour weekly cap:

- Capacity = 3 × 20 = **60 staff-hours**.
- Deficit = **46 staff-hours**, so no valid schedule exists.

That is intentional: the file keeps the schema readable while exercising the infeasibility path. Use `team_synthetic_12.csv` (12 × 20 = 240 staff-hours of capacity) when you need a feasible instance.

Note that 106 staff-hours is the theoretical demand. With an 11-hour operating day, two-person coverage, and an 8-hour maximum shift, each day needs at least four shifts. A valid pattern is two short morning shifts plus two long day shifts, for example `08:00–11:00` and `11:00–19:00`. The validator performs a shift-aware check, not only the aggregate capacity check.

## Invalid fixtures

| File | Expected diagnostic |
|---|---|
| `invalid/duplicate_names.csv` | `E-NAME-002` — `Alice` and `alice` collide. |
| `invalid/missing_name.csv` | `E-NAME-001` — blank name with data in the row. |
| `invalid/reversed_interval.csv` | `E-TIME-002` — start is not before end. |
| `invalid/bad_time_format.csv` | `E-TIME-001` — `25:00` is not a valid time. |
| `invalid/wrong_field_count.csv` | `E-CSV-005` — 12 fields against a 10-column header. |

## Warning fixtures

| File | Expected diagnostic |
|---|---|
| `warnings/misaligned_time.csv` | `W-TIME-001` — `09:10-10:20` snapped to `09:00-10:30`. |
| `warnings/outside_operating_hours.csv` | `W-TIME-002` — `06:00-07:00` is outside operating hours. |
| `warnings/desired_over_cap.csv` | `W-HOURS-001` — 30 desired hours exceeds the 20-hour cap. |
| `warnings/full_day_busy.csv` | `W-AVAIL-001` — buffered busy time covers the whole Monday. |
| `warnings/duplicate_tag.csv` | `W-TAG-002` — `Mentor` duplicates `mentor`. |
| `warnings/unknown_column.csv` | `W-SCHEMA-001` — `Notes` is ignored. |

## Privacy

- All names and times here are invented.
- The importer must keep this data in memory only.
- Do not add real schedules, real names, or exported production files to this folder.
