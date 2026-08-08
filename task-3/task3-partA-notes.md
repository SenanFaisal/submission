# Task 3 Part A — Notes

Used https://open.er-api.com/v6/latest/{base} (no key required). Response shape confirmed by
calling it directly: `{ result, base_code, rates: { ... }, time_last_update_utc, ... }`.

## What each folder checks and why

**1 — Data-driven base currency checks** (driven by `fx-rates-data.csv`: USD, EUR, GBP, PKR, SAR)
Runs the same request once per base currency instead of duplicating it five times. For each one:
status 200, response time under 3s (a rate lookup shown live during a transfer confirmation
screen needs to be fast, not just eventually correct), the response actually has `rates` and
`base_code` matching what was requested, and the currencies a transfer flow would realistically
need (USD/EUR/GBP/PKR/SAR) are present as positive numbers within a sane range. The upper
bound (<1,000,000) isn't a real FX ceiling — it's a guard against the API returning corrupted or
placeholder values for a currency, which would otherwise pass a bare "is it a number" check.

**2 — Chained request**
Captures the USD→EUR rate from one call and checks it against 1 ÷ (EUR→USD rate) from a
second call. Reason for choosing this value specifically: it's the cheapest way to catch an
internally inconsistent rate table (e.g. stale caching on one currency pair but not another)
without needing a second provider to compare against. A small tolerance (0.01) is allowed since
the two calls can land on slightly different rate snapshots.

**3 — Negative case**
Requests `/latest/ZZZ`. Asserts the body returns an explicit `result: "error"` rather than
silently falling back to default data. Deliberately does NOT hard-assert a specific HTTP status
code, because I could not confirm ahead of time whether this provider returns that error body
under HTTP 200 or a 4xx — the test logs the actual status and accepts a few plausible values.
**Run this and check the console log — if it comes back as 200, write that up as a finding**:
a rates provider returning a body-level error under a 200 status is a real integration risk for a
banking backend that checks status codes before parsing the body, and is exactly the kind of
"is this what I'd want from a rates provider in a banking app" judgment Part A asks for.
