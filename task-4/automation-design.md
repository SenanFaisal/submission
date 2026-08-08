# Task 4 — Automation Design (Bank Operations Console)

Note: this design is based on the written scenario description. The attached
`Bank_Admin_Console_Demo.html` should be treated as the authoritative reference for exact
DOM/attribute details — if anything below conflicts with what the file actually shows, the file
wins, and the discrepancy is worth raising in the technical interview.

## 1. Project structure & Page Object Model

The trap is one giant `AdminConsolePage` class with a method per field. The fix is to model
the *field* as the reusable unit rather than "the page," since every field shares the same
wrapper structure regardless of tab.

```
tests/
  admin-console/
    admin-console.spec.ts
    kyc-thresholds.spec.ts
    transfer-limits.spec.ts
    ...
pages/
  admin-console/
    AdminConsolePage.ts        // owns tab navigation only
    tabs/
      OnboardingTab.ts
      KycThresholdsTab.ts
      TransferLimitsTab.ts
      FeesTab.ts
      NotificationsTab.ts
      ChannelsTab.ts
      AuditSettingsTab.ts
    components/
      ConfigField.ts           // the reusable unit - one wrapper, any control type
fixtures/
  admin-console.fixtures.ts
data/
  admin-console-test-data.ts
```

`AdminConsolePage` only selects a tab and returns a tab object. Each `TabXxx` class just lists
which field labels exist on that tab — no interaction logic. All interaction logic lives once,
in `ConfigField`. A tab class ends up ~15-20 lines, not hundreds. Adding a 5th tab means adding
one small class listing its fields, without touching interaction logic at all.

## 2. Locator strategy

Two separate problems:

**Finding the wrapper** — by label text or the stable descriptive attribute, never the dynamic
`slot="field-N"`:
```
getFieldWrapper(labelText) {
  return page.locator('[data-field-wrapper]')
    .filter({ has: page.getByText(labelText, { exact: true }) });
}
```
This is the one locator allowed to be durable across the session — anchored to the label, which
the requirement guarantees doesn't change.

**Handling three control types behind one interface** — `ConfigField` exposes one method per
intent (`setValue`, `getValue`, `toggle`), and detects the control type relative to the wrapper
it already found — never by searching the page for the dynamic identifier directly:

```
class ConfigField {
  constructor(private wrapper: Locator) {}

  async setValue(value: string | boolean) {
    const input = this.wrapper.locator('input[type=text], input:not([type])');
    const checkbox = this.wrapper.locator('input[type=checkbox]');
    const dropdown = this.wrapper.locator('select, [role=listbox]');

    if (await checkbox.count()) return checkbox.setChecked(value as boolean);
    if (await dropdown.count()) return dropdown.selectOption(value as string);
    return input.fill(value as string);
  }
}
```
The dynamic identifier is never referenced anywhere — it can change every session because
nothing depends on its value. The wrapper is the anchor; the control type is detected, not
assumed.

## 3. Handling dynamic DOM

Tab switches/scrolling replacing elements means any locator held across a tab switch is
potentially stale — so the rule is: never hold a locator across a tab switch. Each
`TabXxx.getField()` re-resolves the wrapper fresh, right before use. Playwright locators
re-query on each action by design, which is why it's the framework choice here rather than a
tool that hands back a live element reference (a raw Selenium `WebElement` is the classic
staleness trap).

For scrolling: `scrollIntoViewIfNeeded()` on the wrapper immediately before interacting with it,
every time — not scroll-once-and-assume-it-stays-visible.

What NOT to do: hard-coded `sleep()`/`wait(2000)` calls to "let the tab settle" — the most
common way these suites end up both slow and still flaky. Wait on a condition instead — the
first field of the newly active tab being visible and stable — before interacting with anything
else on that tab.

## 4. Test data strategy

No hard-coded values, a shared environment, and no on-demand DB reset shape this more than
anything else:

- Field values live in a data file per tab, not inline in the spec.
- Anything that must be unique per run (a threshold name, a channel label) gets a run-scoped
  suffix (timestamp or short random ID) generated at test start, so parallel runs and other
  teams' concurrent usage of the shared environment don't collide.
- Since the DB can't be reset on demand, the suite cleans up after itself: each test that
  changes a named config item records what it changed and reverts it in a teardown step, not
  just at the end of the whole suite — so a mid-run failure doesn't leave the environment dirty
  for the next run.
- Where a field's current value matters (a threshold other tests might also touch), read it
  before changing it and restore that exact value afterward, rather than assuming a fixed
  default.

## 5. Reusability & scalability

Adding a 5th tab: one `TabXxx` class listing that tab's field labels, plus a data-file entry —
nothing in `ConfigField`, `AdminConsolePage`, or the wait/locator logic changes. Adding a 20th
field in an existing tab is a one-line addition to that tab's field list.

Scaling from 7 tabs to 20: group tests so tabs run in parallel workers (each tab's tests are
independent, operating on different config sections), keeping wall-clock time roughly flat as
tab count grows. CI runs a fixed smoke subset on PR and the full set nightly, with results
published to a dashboard rather than raw CI log output, since 20 tabs of field-level tests is
enough volume that a flat pass/fail log stops being readable.

## 6. Tooling

Playwright with TypeScript. Built-in auto-waiting and locator re-resolution directly address the
dynamic-DOM/staleness problem in this brief, without a custom wait-and-retry layer on top.

Trade-off accepted: Playwright's ecosystem and troubleshooting base is smaller than Selenium's,
and if the rest of the bank's stack is Java-only, introducing a Node/TS tool means a separate CI
runner and dependency chain — a real integration cost a Selenium-Java choice wouldn't carry.
Selenium would be the right call instead if the team's existing automation is already
Java-based and that consistency matters more than the auto-waiting benefit.
