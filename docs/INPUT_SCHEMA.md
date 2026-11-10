# ShiftOpt — Input Schema (MVP)

**Version:** 0.3
**Status:** accepted for the MVP
**Scope:** employee import only. Workplace settings, scheduling constraints, and their defaults live in `docs/SCHEDULING_RULES.md`.

## 1. Entry methods

Two entry methods share the same internal model:

- **CSV import (MVP):** one spreadsheet-exported CSV, one employee per row.
- **Calendar editor (later):** add employees and mark busy periods in the UI, and edit imported data before generating schedules.

Employee self-service links are **not part of the MVP** because they require a secure way to transfer data between users.

The CSV carries employee data only. Operating hours, staffing targets, hour caps, shift rules, and buffers are workplace settings: they come from the defaults in `docs/SCHEDULING_RULES.md` and are editable in the UI.

## 2. File format

| Property | Rule |
|---|---|
| Encoding | UTF-8. A UTF-8 BOM is accepted and stripped (`W-CSV-001`). |
| Delimiter | Comma (`,`). Semicolon-delimited files are rejected with a hint (`E-CSV-004`). |
| Line endings | LF or CRLF. |
| Quoting | RFC 4180. Quoted cells may contain commas, semicolons, and line breaks. |
| Size limit | 1 MB and 500 employee rows. |
| Header | Required. Names are matched case-insensitively after trimming and collapsing whitespace. |
| Column order | Irrelevant. |
| Unknown columns | Ignored with warning `W-SCHEMA-001`. |
| Duplicate columns | Rejected (`E-CSV-006`). |
| Fully blank rows | Skipped silently. |

Filename: `team.csv` (the name is a convention, not a requirement).

## 3. Columns

| Column | Required? | Example |
|---|---|---|
| `Name` | Yes | Alice |
| `Desired Hours` | No | 12 |
| `Tags` | No | mentor; first-aid |
| `Monday Busy` | No | 09:00-10:15; 13:00-14:00 |
| `Tuesday Busy` | No | 11:00-12:30 |
| `Wednesday Busy` | No | |
| `Thursday Busy` | No | |
| `Friday Busy` | No | |
| `Saturday Busy` | No | |
| `Sunday Busy` | No | |

Only `Name` is mandatory. A missing weekday column means "no busy times supplied for that day". If none of the seven weekday columns is present, the import emits `W-SCHEMA-002` and treats every employee as fully available during operating hours.

## 4. Field rules

### 4.1 Name

- Trimmed, internal whitespace collapsed, Unicode NFC normalized.
- 1–80 characters after normalization.
- Unique within the file, case-insensitive (`Alice` and `alice` collide).
- The original spelling is kept for display; internal IDs (`e01`, `e02`, …) are generated after validation.

### 4.2 Desired Hours

- Decimal number in the range 0–168.
- Blank or `0` means "no personal hour target".
- A value above the workplace weekly cap is accepted with warning `W-HOURS-001`; the cap still wins.
- This is a soft target, never a hard constraint.

### 4.3 Tags

- Separated by semicolons: `mentor; first-aid`.
- Trimmed, empty entries dropped, duplicates removed case-insensitively (`W-TAG-002`).
- At most 10 tags, each at most 32 characters.
- Case is preserved for display; matching is case-insensitive.

### 4.4 Busy cells

An empty cell means **no known conflict** during operating hours, not that the employee is unavailable.

Grammar:

```
cell      := interval ( ";" interval )*
interval  := time separator time
time      := HH ":" MM | H ":" MM
separator := "-" | "–" | "—" | " to "
HH        := 0..24   (24 only as an end time, with MM = 00)
MM        := 00..59
```

Rules:

- Whitespace around tokens is ignored: `09:00 - 10:15` is valid.
- At most 20 intervals per cell.
- `start` must be strictly before `end`; the end is exclusive.
- Overnight intervals (end on the next day) are not supported in the MVP. An interval that appears to cross midnight, such as `22:00-02:00`, is reported as `E-TIME-003`; other reversed intervals are reported as `E-TIME-002`.
- Busy periods repeat weekly.
- Intervals may lie outside operating hours; they are clipped for scheduling and reported with `W-TIME-002` when part of the interval is removed.
- Overlapping or adjacent intervals are merged after parsing.
- Intervals that become adjacent after buffering are merged again.

### 4.5 Time grid

The MVP schedules on a configurable grid (default 15 minutes). Imported times that are not aligned are **snapped conservatively**, not rejected:

- start rounds **down** to the previous grid line;
- end rounds **up** to the next grid line;
- a warning `W-TIME-001` reports the original and normalized values.

Example with a 15-minute grid: `09:10-10:20` becomes `09:00-10:30`. Rounding outward keeps the employee unavailable slightly longer, so the solver can never over-book them.

A 15-minute travel/preparation buffer (configurable) is applied before and after every busy interval, then the result is clipped to operating hours. If the buffered interval covers the whole operating day, the employee is marked unavailable that day (`W-AVAIL-001`).

## 5. Validation

The importer collects **all** diagnostics in one pass and rejects the whole file if any error is present. There is no partial import. Warnings do not block import but are shown to the supervisor.

Each diagnostic contains: record number (header = record 1), column name, code, message, and the offending value.

| Code | Severity | Condition |
|---|---|---|
| `E-CSV-001` | error | File is not valid UTF-8. |
| `E-CSV-002` | error | File exceeds 1 MB or 500 employee rows. |
| `E-CSV-003` | error | Header is missing `Name`. |
| `E-CSV-004` | error | Header appears to be semicolon-delimited (no commas in the header line). |
| `E-CSV-005` | error | A record has a different number of fields than the header. |
| `E-CSV-006` | error | Duplicate column name. |
| `E-NAME-001` | error | Empty name. |
| `E-NAME-002` | error | Duplicate name (case-insensitive). |
| `E-NAME-003` | error | Name longer than 80 characters. |
| `E-HOURS-001` | error | `Desired Hours` is not a number. |
| `E-HOURS-002` | error | `Desired Hours` is outside 0–168. |
| `E-TAG-001` | error | More than 10 tags, or a tag longer than 32 characters. |
| `E-TIME-001` | error | Interval syntax is invalid. |
| `E-TIME-002` | error | Interval start is not before end. |
| `E-TIME-003` | error | Overnight interval. |
| `E-TIME-004` | error | More than 20 intervals in one cell. |
| `W-TIME-001` | warning | Interval snapped to the time grid. |
| `W-TIME-002` | warning | Interval clipped to operating hours. |
| `W-AVAIL-001` | warning | Employee unavailable for the whole day after buffering. |
| `W-HOURS-001` | warning | Desired hours exceed the weekly cap. |
| `W-TAG-002` | warning | Duplicate tag removed. |
| `W-SCHEMA-001` | warning | Unknown column ignored. |
| `W-SCHEMA-002` | warning | No weekday columns found. |
| `W-CSV-001` | warning | UTF-8 BOM removed. |

Validation also runs the feasibility pre-checks in `docs/SCHEDULING_RULES.md` §5. Those checks produce `E-FEAS-*` and `W-FEAS-*` diagnostics when the workplace configuration and the imported availability cannot satisfy the mandatory rules.

## 6. Normalization pipeline

1. Read bytes, decode UTF-8, strip BOM, enforce size limits.
2. Parse with a standard CSV parser; normalize the header.
3. Parse each record; check field count, names, desired hours, and tags.
4. Parse busy cells into minute intervals.
5. Sort and merge overlapping or adjacent intervals.
6. Snap to the configured grid (round outward) and emit warnings.
7. Apply the buffer, merge again, and clip to operating hours.
8. Compute per-day free slots and total available minutes per employee.
9. Generate internal employee IDs in file order.
10. Run validation and feasibility pre-checks; return either a normalized model or an error list.

## 7. Internal representation

The parser returns a normalized, frontend-independent model. Times are minutes from midnight; the solver adapter converts them to slot indices.

```json
{
  "schema_version": "0.3",
  "resolution_minutes": 15,
  "workplace": {
    "operating_hours": {
      "monday":    { "open": 480, "close": 1140 },
      "tuesday":   { "open": 480, "close": 1140 },
      "wednesday": { "open": 480, "close": 1140 },
      "thursday":  { "open": 480, "close": 1140 },
      "friday":    { "open": 480, "close": 1020 },
      "saturday":  null,
      "sunday":    null
    },
    "staffing_windows": [
      { "days": ["monday", "tuesday", "wednesday", "thursday"],
        "start": 480, "end": 1140, "target": 2, "minimum": 2 },
      { "days": ["friday"],
        "start": 480, "end": 1020, "target": 2, "minimum": 2 }
    ],
    "max_weekly_hours": 20
  },
  "employees": [
    {
      "id": "e01",
      "name": "Alice",
      "desired_weekly_minutes": 720,
      "tags": ["mentor"],
      "busy": { "monday": [[540, 615], [780, 840]] },
      "unavailable": { "monday": [[525, 630], [765, 855]] },
      "available_minutes": 2985
    }
  ],
  "diagnostics": {
    "errors": [],
    "warnings": [
      {
        "record": 2,
        "column": "Monday Busy",
        "code": "W-TIME-001",
        "message": "Interval 09:10-10:20 snapped to 09:00-10:30.",
        "value": "09:10-10:20"
      }
    ]
  }
}
```

`busy` keeps the parsed intervals for traceability; `unavailable` is the buffered, merged, clipped version used for scheduling.

## 8. Privacy

- The file is parsed in memory for the current session only.
- No upload, no disk write, no `localStorage`/`sessionStorage`, and no analytics payload contains employee data.
- The UI must disclose where processing happens before the first import.
- Exported schedules are generated only on explicit user request.
- Public examples and tests use synthetic names only.

## 9. Examples

Valid row:

```csv
Name,Desired Hours,Tags,Monday Busy,Tuesday Busy,Wednesday Busy,Thursday Busy,Friday Busy,Saturday Busy,Sunday Busy
Alice,12,mentor,09:00-10:15; 13:00-14:00,11:00-12:30,,,,,,
```

Invalid row (duplicate name and reversed interval):

```csv
alice,12,mentor,10:00-09:00,,,,,,
```

See `examples/README.md` for the full set of valid, warning, and invalid fixtures.

## 10. Deferred

- Preferred shifts in CSV. Planned as optional `Preferred <Day>` columns using the same interval grammar, meaning "preferred working windows" (soft). The parser will ignore them with a warning until implemented.
- Inline calendar editing and preferred-time selection.
- Optional Excel workbook import (`.xlsx`).
- Secure self-service availability collection, subject to a separate privacy design.
- More flexible day/time resolution and configurable presets.

## 11. Change log

| Version | Change |
|---|---|
| 0.1 | Two CSV files: `employees.csv` and long-format `busy_times.csv`. |
| 0.2 | Single matrix CSV, one row per employee, one column per weekday. |
| 0.3 | Detailed file, field, time, and validation rules; conservative snapping; error/warning codes; internal model; workplace defaults moved to `SCHEDULING_RULES.md`. |
