# QA Technical Assessment — Submission

Level assessed against: Senior (Tasks 1–4 required, Task 5 optional bonus).

## Time spent

| Task | Suggested | Actual |
|---|---|---|
| Task 1 — Test Design & Risk Coverage | 40 min | 40 min |
| Task 2 — Exploratory Testing & Bug Reporting | 25 min | 30 min |
| Task 3 — API Testing | 30 min | 35 min |
| Task 4 — Automation Design | 35 min | 35 min |
| Task 5 — Quality Strategy (optional bonus) | 30 min | Not attempted — see below |
| Packaging (repo, README, evidence, ai-usage.md) | 20–30 min | 25 min |
| **Total** | ~2–2.5 hrs | ~2 hrs 45 min |

Task 2 ran slightly over because ParaBank (primary site) was returning a Bad Gateway error, so
time went into confirming that, switching to the documented backup (demo.testfire.net), and
establishing a clean baseline account on a shared/already-corrupted demo environment before
testing could start properly. Task 3 ran slightly over confirming the actual response shape and
error behavior of the live API before writing assertions against it.

## Assumptions and questions for the BA/PO

See `task-1/test-design-risk-coverage.md` section 1.1 for the full list with reasoning. In
short: currency scope, exact limit reset semantics, OTP retry/expiry rules, what AC6's "cannot
be completed" actually covers, idempotency/duplicate-submission handling, beneficiary/account
state changes mid-flow, fee disclosure timing, and behavior when a debit succeeds but
confirmation fails partway. Where I couldn't get an answer, I stated the assumption I tested
against explicitly in that section rather than guessing silently.

## What was deliberately left out, and why

- **Task 5** was not attempted — it's an optional bonus at Senior level, and time was prioritized
  on the four required tasks instead.
- **Task 2 / Bill Pay comparison**: the assessment's exploration notes suggested comparing
  Transfer Funds and Bill Pay validation. The backup site used (AltoroMutual, since ParaBank was
  down) has no Bill Pay feature at all — confirmed via its own menu and REST API docs. Compensated
  by testing Transfer Funds more thoroughly instead.
- **Task 1's 12-case cap**: multi-currency/FX scenarios, accessibility, and a full negative-input
  matrix on the amount field were left out of the 12 test cases — reasoning is in section 1.4.
- **Task 3**: only USD/EUR/GBP/PKR/SAR were used as target currencies for schema/sanity checks,
  not the full currency list, since the assessment is about validation approach, not exhaustive
  coverage. The exact HTTP status code for the malformed-currency negative case was left as an
  open, logged finding rather than a hard assertion — noted in `task-3/task3-partA-notes.md`.
- **Task 4**: designed from the written scenario description; if the attached
  `Bank_Admin_Console_Demo.html` differs from what's described, that discrepancy should be
  raised and discussed in the technical interview rather than silently reconciled here.

## Folder structure

```
task-1/   Test design & risk coverage
task-2/   Exploratory testing & bug reports (+ evidence/)
task-3/   Postman collection, environment, data file, and written API analysis
task-4/   Automation design (no code, as required)
ai-usage.md
```
