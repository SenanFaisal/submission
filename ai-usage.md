# AI Usage

Used Claude (chat) throughout for structuring answers and drafting, and Claude in Chrome for
part of the Task 2 exploratory testing. Details below, as requested.

## Prompts used (representative sample)

**Task 1:**
> "Here's the requirement as received from the BA [pasted the AC1-AC6 requirement]. What
> questions should I raise before testing begins, and what's the highest-risk area of this
> feature?"

**Task 2:**
> "Give me a prompt so Claude in Chrome can do exploratory testing on the AltoroMutual demo
> banking site — functional bugs only, no security testing, need 2 distinct root-cause bugs
> with full repro steps and evidence."

**Task 3:**
> "Build a Postman collection for the open.er-api.com FX rates endpoint — status/schema/response
> time validation, data-driven across multiple base currencies via CSV, a chained request that
> captures and reuses a value, and a negative case for a malformed currency code."

**Task 4:**
> "Design (no code) an automated regression approach for an admin console with tabs, three
> control types per field, and a dynamic per-session identifier that shouldn't be used as the
> primary locator. Cover project structure, locator strategy, dynamic DOM handling, test data
> strategy, scalability, and tooling trade-offs."

## What was kept vs. rejected

- **Kept, largely as drafted:** the Postman collection structure and test assertions (Task 3),
  and the Task 4 automation design — these matched my own reasoning about how I'd approach the
  problem, so I reviewed and kept them with light edits.
- **Rewritten in my own words:** the Task 1 and Task 2 write-ups. The AI's first drafts were
  directionally right but read too polished/generic in places, so I rewrote sections to reflect
  how I'd actually phrase things and to match what I could personally verify.
- **Rejected/overridden:** an early draft severity rating on one of the Task 2 bugs was
  adjusted down after I re-checked it myself — the AI's first pass treated a rounding-display
  mismatch as more severe than I judged it to be once I confirmed no money was actually
  lost or duplicated in that case.
- **Verified independently, not just accepted:** both Task 2 bugs were reproduced manually by me
  before being written up, following Claude in Chrome's initial findings — the repro steps,
  screenshots, and final wording in `task-2/bug-report.md` reflect my own run-through, not the
  raw AI output.
