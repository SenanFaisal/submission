# Task 3 Part B — POST /api/v1/payments

## 3.1 Highest-risk scenarios

What makes this different from a normal CRUD endpoint is that a "successful" response and a
"correct" response aren't the same thing — a 201 tells you the API accepted the request, not
that money actually moved the way it should have. So the highest-risk scenarios are the ones
where the response looks fine but the underlying money movement is wrong, duplicated, or stuck
in between states.

**The amount object (value, currency, precision):**
- Sending a value with more decimal places than the currency supports (e.g. `1500.755` for SAR,
  which — like most currencies — trades in 2 decimal places) and checking whether the API
  rejects it or silently truncates/rounds it without telling the caller.
- Zero and negative values — should be rejected outright, not processed as a no-op or a reverse
  transfer.
- A currency the source account doesn't actually hold, or a currency not supported at all —
  does the API validate against the account's actual currency, or just that the string is
  well-formed?
- Extremely large values (integer overflow / precision loss territory for whatever numeric type
  the backend uses to store `value`) — floating point storage of money is a classic source of
  off-by-a-cent bugs at scale.
- Mismatched precision between the value and what `fee` ends up being calculated on — if fee is
  computed from a rounded value but debited from an unrounded one, the ledger won't reconcile.

**Limit and insufficient-funds paths:**
- The boundary itself — exactly at the limit, one cent over, one cent under — not just "clearly
  over the limit," since off-by-one errors live at boundaries.
- Two requests submitted close together that are each individually within the limit but combined
  exceed it (the race condition version of the limit check, not just the single-request version).
- What `422 Limit Exceeded` actually contains — does it tell the caller which limit (daily vs
  per-transaction) so the client can show a useful message, or is it generic?
- Whether `402 Insufficient Funds` is checked against the *current* available balance including
  any funds already reserved by other pending transfers, not a stale balance read at the start of
  the request.
- What happens if the balance check passes but the account balance changes (another transfer
  clears) in the narrow window before the debit is committed.

## 3.2 Idempotency-Key

The Idempotency-Key exists so that a client retry — after a timeout, a dropped connection, or a
mobile app that doesn't know if its last request actually landed — doesn't cause the payment to
be executed twice. The client generates one key per logical payment attempt and sends the same
key on every retry of that same attempt.

**How I'd test it:**
- Send the same request twice with the same Idempotency-Key and confirm only one payment is
  created — the second response should return the *same* `paymentId`/`reference` as the first,
  not a new one and not an error.
- Send the same Idempotency-Key with a *different* body (different amount or beneficiary) and
  confirm the API rejects it (409 Conflict is the obvious fit) rather than silently processing
  the new body under the old key, or silently returning the old result for a different request.
- Send two requests with the same key at effectively the same time (concurrent, not sequential)
  to check the server actually locks on the key rather than letting both through in a race.
- Reuse a key from a genuinely completed payment after enough time has passed to see if the key
  is ever expired/recycled, and whether that's documented.

**Retry after a timeout or 503:** the client doesn't know if the original request succeeded
server-side or not, so the correct behavior is: retry with the *same* Idempotency-Key. If the
original request never reached the point of creating a payment, the retry creates it once. If it
did create a payment but the response was lost in transit, the retry should return the existing
result rather than creating a duplicate. This only works if idempotency is enforced at the point
where the payment record is created, not just at the API gateway layer — which is worth
confirming rather than assuming.

## 3.3 Verifying money actually moved (beyond the 201)

- Query the source account's balance/ledger directly (via a separate accounts endpoint or DB
  check, not the payments API) and confirm it decreased by exactly `value + fee`, not just that
  *a* transaction appeared.
- Confirm a corresponding entry exists on the beneficiary side — the debit and credit should be
  two sides of the same movement, not just "our side went down."
- Check the transaction actually reaches its final state. `status: PENDING` at 201 doesn't mean
  done — poll or subscribe to whatever mechanism reports final settlement, and verify the eventual
  state is `COMPLETED` (or a documented failure state), not left hanging indefinitely.
- Reconcile the `reference` returned in the response against whatever the downstream ledger/audit
  system records as the reference for that movement — a mismatch there means the API and the
  system of record have drifted, even if both individually "worked."
- Re-run the same balance check after a deliberate delay to catch any asynchronous correction or
  reversal that might happen after the initial success response.
