# Task 2 — Exploratory Testing & Bug Reporting

**Site availability note:** The primary site, parabank.parasoft.com, returned a Bad Gateway
error at the time of testing. Used the documented backup, https://demo.testfire.net/
(AltoroMutual) instead, as instructed.

**Scope note:** AltoroMutual has no self-registration flow and no Bill Pay feature (confirmed
via the app's menu and its own REST API docs at /swagger/index.html, which lists only Login,
Account, Transfer, Feedback, Admin, and Logout). Signed in using `jsmith` / `demo1234` —
AltoroMutual's publicly documented demo account, not a credential obtained via any bypass. Bill
Pay comparison testing (planned per the exploration approach) could not be run since the feature
doesn't exist in this build; Transfer Funds was tested more thoroughly to compensate.

**Environment note:** this is a shared public demo instance. Several accounts already showed
corrupted balances (e.g. values like -1E+20) from other testers/scanners before testing began.
Account 800013 Checking (clean $400.00 starting balance) was used as the primary test account,
with 800011 Checking as the transfer counterparty, so that every balance change could be
attributed to actions taken during this test.

## Bug 1 — Transfer Funds has no balance check; account goes negative with no warning

**Environment:** demo.testfire.net, Chrome, logged in as jsmith.

**Preconditions:** Account 800013 Checking starting balance $400.00; account 800011 Checking
exists as destination.

**Steps to reproduce:**
1. Log in at demo.testfire.net/login.jsp with jsmith / demo1234.
2. Go to Transfer Funds, From = 800013, To = 800011.
3. Enter 99999 as the amount, click Transfer Money.
4. Open Account Activity for 800013.

**Actual result:** Transfer confirms as successful ("99999.0 was successfully transferred from
Account 800013 into Account 800011..."), no error anywhere. 800013's balance drops to
-$99,599.00; 800011 gains the full $99,999.00.

**Expected result:** A transfer above available balance should be rejected with an
insufficient-funds error, or at minimum enforce a defined overdraft limit — not silently confirm
and let the account go arbitrarily negative.

**Severity / Priority:** Critical / P1 — this is the core control on a money-movement feature,
missing entirely.

**Evidence:** Screenshot of the success confirmation message; screenshot of Account Activity
showing the -$99,599.00 balance and the withdrawal line.

## Bug 2 — Confirmation message shows the entered amount, not the amount actually recorded

**Environment:** demo.testfire.net, Chrome, logged in as jsmith.

**Preconditions:** 800013 Checking and 800011 Checking both accessible (balance state doesn't
matter — the defect isn't balance-dependent).

**Steps to reproduce:**
1. Go to Transfer Funds, From = 800013, To = 800011.
2. Enter 10.999 (three decimals) as the amount, click Transfer Money.
3. Read the confirmation message.
4. Open Account Activity for both accounts and compare against the confirmation text.

**Actual result:** Confirmation reads "10.999 was successfully transferred..." but Account
Activity on both sides shows the transaction posted as $11.00 — the system rounded the value for
the ledger but never updated what it told the user.

**Expected result:** Either reject anything beyond 2 decimal places at input time, or if
rounding is intentional, show the rounded value consistently everywhere the user sees it. Here
the confirmation and the actual record disagree.

**Severity / Priority:** Medium / P2 — no money is actually lost or duplicated, but a
user-facing number not matching the system-of-record is a real correctness defect, just lower
stakes than Bug 1.

**Evidence:** Confirmation page text; Account Activity screenshots from both accounts showing
the $11.00 postings.

## 2.2 Approach

Started by identifying which account had a trustworthy balance, since this shared public demo
had clearly been hammered by other testers already — some balances were in the octillions.
Testing against a known $400.00 baseline made every balance change attributable to my own
actions. Went after Transfer Funds first as the highest-risk money-movement path, specifically
trying to push it past its stated limit before checking anything cosmetic — the over-balance
issue surfaced immediately. From there, checked whether input formatting was handled
consistently, since that's a common place demo apps cut corners, and found the rounding
mismatch. Negative amounts, zero, and same-account transfers were all correctly blocked
client-side, so those weren't pursued further.

With two more hours: cross-account-type transfers (savings to credit card, if available),
sub-cent amounts near zero, and whether balance figures stay consistent across concurrent
sessions.
