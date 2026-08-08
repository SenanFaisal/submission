# Task 1 — Test Design & Risk Coverage

## 1.1 Questions and Assumptions

Before testing begins, these are the questions I'd raise with the BA/PO, and why each one
matters:

1. **Currency scope** — Is this domestic-currency-only, or can source/beneficiary accounts hold
   different currencies? If FX is in scope, AC3 (fees) and AC4 (limits) both need a currency
   dimension, and the risk profile of the whole feature changes.
2. **Limits: 20,000 / 10,000 — per what, and reset when?** Daily rolling 24h or calendar day?
   Does it count only this transfer type or all outgoing transfers? Wrong reset logic is a
   classic real-world banking bug that lets customers exceed intended exposure.
3. **OTP failure/retry behavior** — how many attempts, what happens on expiry, is there a resend
   limit, and is the transfer request "reserved" while waiting for OTP? Directly affects
   double-debit and race-condition risk.
4. **What does "cannot be completed" cover in AC6?** Insufficient funds, beneficiary
   deactivated mid-flow, downstream timeout, duplicate submission — same generic error or
   different messaging per case? A generic error for "insufficient funds" vs "system down" is a
   support-cost and trust issue.
5. **Idempotency / duplicate submission** — what happens if the customer double-taps "Confirm"
   or the app retries after a timeout? This is the single highest-impact gap in a money-movement
   flow, and nothing in the AC addresses it.
6. **Beneficiary/account state at time of transfer** — can a beneficiary or source account be
   edited/deleted/frozen between selection and confirmation? Stale-state risk in a multi-step
   flow.
7. **Fee display and debit** — is the fee shown before confirmation, and is it deducted from the
   transfer amount or charged separately? Undisclosed fee handling is a regulatory/complaint
   risk in banking.
8. **Partial failure after debit** — if the source account is debited but the
   confirmation/SMS step fails, is the transaction reference still generated and reconciled?
   This is the "money left but nothing confirms it" scenario — the worst-case customer
   experience.

Where I can't get an answer in time, my assumption (stated explicitly so it can be challenged):
same-currency transfers only, daily limit resets at midnight in the account's local timezone,
OTP has a max of 3 attempts with a 5-minute expiry, and a debit is never left unconfirmed — the
system always reconciles or reverses it.

## 1.2 Risk Assessment

Ranked by business impact, not by ease of testing:

1. **Double-debit / duplicate transfer execution — highest risk.** A network retry, double-tap,
   or app crash after OTP confirmation could execute the transfer twice. Impact: real financial
   loss to the customer without them initiating it twice — direct harm, regulatory complaint,
   refund cost. For us as delivery partner, this is the kind of defect that ends up in a
   post-incident report with our name on it.
2. **Limit enforcement gaps (AC4).** Limits checked client-side only, or not aggregated
   correctly across near-simultaneous transfers (a race where two transfers, each individually
   under the limit, both get approved). Impact: a control/compliance failure for the bank, not
   just a UX bug.
3. **Debit without confirmation (AC5/AC6 boundary).** Source account debited but reference
   generation, confirmation screen, or SMS fails afterward — customer sees an error while money
   is already gone. Impact: erodes trust immediately, generates manual reconciliation and
   support load.
4. **OTP bypass or weak enforcement (AC2).** A race between "confirm" and "verify OTP" calls,
   or client-side-only OTP validation. Impact: the one authentication gate this AC relies on
   being bypassable is severe, but this is ranked below duplicate-submission and limits because
   it needs a specific flaw to manifest, while those two are close to guaranteed under normal
   real-world conditions (flaky networks, impatient users).
5. **Incorrect fee application (AC3).** Fee not applied, applied twice, or using stale
   configuration. Impact: compounds across transaction volume, reconciliation/audit exposure.
6. **Stale beneficiary/account data used mid-flow.** Funds sent to an invalid or unintended
   destination after a beneficiary/account change mid-flow — hard to reverse in banking.

## 1.3 Test Cases (12)

| # | Title | Precondition | Steps (brief) | Expected Result |
|---|---|---|---|---|
| TC01 | Successful transfer within limits | Source account has sufficient balance, beneficiary registered | Select account → beneficiary → enter amount ≤10,000 → confirm OTP | Account debited, fee applied, ref number generated, confirmation shown, SMS sent |
| TC02 | Transfer rejected above per-transaction limit | Balance sufficient | Enter amount = 10,000.01 | Blocked before OTP step, clear limit error shown |
| TC03 | Transfer rejected when daily cumulative limit exceeded | Customer already transferred 15,000 today | Attempt second transfer of 6,000 | Blocked, error references daily limit, not per-transaction limit |
| TC04 | Concurrent transfers near daily limit | Remaining daily allowance is exactly 5,000 | Submit two transfers of 5,000 simultaneously from two sessions | Only one succeeds; both are not approved |
| TC05 | Insufficient balance | Balance lower than entered amount | Enter amount > balance, proceed to OTP | Rejected with insufficient-funds message, no OTP sent |
| TC06 | Incorrect OTP entered | Valid transfer request pending confirmation | Enter wrong OTP | Transfer not executed, account not debited, retry allowed within attempt limit |
| TC07 | OTP expires before entry | OTP sent, wait past expiry window | Enter OTP after expiry | Transfer not executed, customer prompted to resend, no partial debit |
| TC08 | Double-submit / retry after timeout | Transfer submitted, OTP confirmed, simulate network timeout on confirm response | Retry the same confirm action | Only one debit occurs, one reference number generated |
| TC09 | Beneficiary removed after selection, before confirmation | Beneficiary selected in step 1 | Delete/deactivate beneficiary from another session, then confirm | Transfer fails cleanly, account not debited |
| TC10 | Fee correctly calculated and disclosed | Standard fee config active | Complete a transfer, check confirmation screen | Fee shown before final confirmation matches configured fee, correct total debited |
| TC11 | Optional purpose note omitted | — | Complete transfer without a purpose note | Transfer proceeds normally, note field blank on confirmation/SMS |
| TC12 | System failure after debit, before confirmation generated | Simulate backend failure right after debit step | Complete transfer, backend fails at confirmation-generation stage | Reference number still generated and reconciled, no silent money loss |

Note on TC12: this is the case I'd flag hardest to the BA — the spec doesn't say what happens
if debit succeeds but the rest of AC5 fails partway. Written against my stated assumption
(money never disappears silently), but it's really a design gap, not just a test gap.

## 1.4 Coverage Decisions

**Deliberately not covered (within the 12-case cap):** multi-currency/FX scenarios (assumed
out of scope per Q1), accessibility, load/performance under concurrent transfers at scale
(touched only as a functional race in TC04, not a load test), and the full negative-input
matrix on the amount field (decimals, negative numbers, injection strings) — those belong in a
larger regression pack, not the 12 that matter most for a first pass.

**Automate vs stay manual:**
- Automate: TC01, TC02, TC03, TC05, TC10, TC11 — deterministic, no timing/concurrency
  dependency, good regression candidates.
- Stay manual initially: TC04, TC06, TC07, TC08, TC09, TC12 — depend on timing, concurrency, or
  simulated backend failure, which need a dedicated test harness/mocking setup before they're
  reliable in CI. Automating them prematurely would give false confidence if timing isn't
  properly controlled.

**How I'd know it's safe to release:** not "all test cases pass." I'd want the
duplicate-submission and limit-race scenarios specifically verified (not just happy path), fee
disclosure confirmed against the actual fee config (not a mock), and an explicit answer from the
BA on the AC5/AC6 partial-failure gap before calling this release-ready.
