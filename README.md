# ShiftOpt

Privacy-first workforce scheduling decision support for supervisors.

**Simple by default, powerful when needed.** Import employee availability, generate several genuinely different schedules, compare their trade-offs, and export the one you choose. The supervisor always makes the final decision.

## Status

Early design stage: the product and input contracts are written; there is no solver or UI code yet.

- Phase 0 — requirements: complete.
- Phase 1 — input design: complete pending review.
- Phase 2 — mathematical formulation: next.

## What it will do

1. Import recurring busy times and optional preferences from a spreadsheet-friendly CSV.
2. Configure operating hours, staffing targets, and shift rules.
3. Generate multiple valid schedules that differ in meaningful ways.
4. Compare coverage, hours, fairness, preferences, and trade-offs.
5. Export the preferred schedule.

No account is required. Imported employee data is processed in memory and is never stored permanently.

## Documents

| Document | Purpose |
|---|---|
| [`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) | Product contract, scope, privacy, success criteria. |
| [`docs/INPUT_SCHEMA.md`](docs/INPUT_SCHEMA.md) | CSV format, field rules, validation codes, normalized model. |
| [`docs/SCHEDULING_RULES.md`](docs/SCHEDULING_RULES.md) | Hard constraints, soft goals, defaults, feasibility checks, relaxation policy. |
| [`docs/OBJECTIVES.md`](docs/OBJECTIVES.md) | Metrics, objective order, weights, and trade-off presentation. |
| [`docs/ALTERNATIVES.md`](docs/ALTERNATIVES.md) | Diversity metrics, selection algorithm, benchmark targets. |
| [`docs/DECISIONS.md`](docs/DECISIONS.md) | Architecture and product decision log. |
| [`examples/README.md`](examples/README.md) | Synthetic fixtures, including intentionally invalid files. |

## Repository layout

```
docs/        Product and technical contracts
examples/    Synthetic CSV fixtures (never real employee data)
src/         Solver and library code (empty for now)
tests/       Automated tests (empty for now)
experiments/ Benchmarks and evaluation runs
AI/          Local working notes, intentionally git-ignored
```

## Privacy

- No account and no permanent storage of imported employee data.
- Prefer local or browser-based processing.
- No automatic saving to browser storage.
- Export only on explicit request.
- Public examples and tests use synthetic names only.

## Development

Python 3.11+ is the intended runtime; the solver will use CP-SAT. Tooling and packaging are not set up yet. Commit after each validated phase; push only when the maintainer asks.
