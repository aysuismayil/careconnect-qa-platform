# BUG-003 — Customer is double-charged when retrying payment after a network timeout

**Title/Summary:**
[Payments] A network timeout during checkout followed by a user-initiated retry results in two separate charges for a single booking, instead of the retried request being treated as idempotent.

**Short Description:**
When the payment confirmation request times out client-side (e.g., due to a dropped connection) but actually succeeds server-side, the customer sees an error and naturally retries. The retry creates a **second, separate charge** for the same booking attempt rather than recognizing the original request already succeeded. This is a direct financial-risk bug — customers can be charged twice for one booking.

**Preconditions:**
- Staging environment with network throttling/interception available (Chrome DevTools "Network" tab, or a proxy tool).
- Customer test account with a valid test payment method.
- A caregiver with an open slot available.

**Steps to Reproduce:**
1. Log in as a customer test account, select an available slot, and proceed to checkout.
2. Open Chrome DevTools → Network tab, and enable request throttling/blocking configured to simulate a dropped connection *after* the request reaches the server but *before* the client receives the response (done in staging by adding an artificial server-side delay + killing the client connection mid-response, coordinated with a backend dev for this specific repro).
3. Submit payment.
4. Observe the client times out / shows a generic error ("Something went wrong, please try again").
5. Click "Pay Now" again (retry) using the same payment details.
6. Check the customer's booking history and the staging database for the number of `payment` records tied to this booking attempt.

**Actual Result:**
Two `payment` records are created, and two separate charges are issued against the same test card for what the customer experienced as a single booking attempt. Only after the second charge does a `confirmed` booking appear — the first (successful but "lost" from the client's perspective) charge exists with no cleanly linked booking, effectively an orphaned charge.

**Expected Result:**
The retried request should be recognized as a retry of the same booking attempt (via an idempotency key generated at the start of checkout and reused on retry) and should not create a second charge. Either the original successful payment/booking should simply be returned to the client, or if a genuine new attempt is needed, the first charge must be reliably voided/reconciled first.

**Test Data:**
- Customer: `cust-retry-qa@example-mail.com`
- Test card: `4242 4242 4242 4242`, exp 12/29, CVC 123
- Booking attempt: Elder Care, 2 hours, $60.00 total
- Simulated delay: artificial 8-second server-side hold on the confirm endpoint combined with a 5-second client timeout, to reliably force a "client gave up, server kept processing" scenario

**Environment/Configuration:**
- Environment: Staging
- Build: `payments-service v3.1.0`, `web-client v4.12.0` (sample version labels)
- Browser: Chrome 126, DevTools network throttling ("Slow 3G" + custom timeout override)
- Payment provider: sandbox/test mode only

**Evidence/Attachments:**
- `evidence/bug-003-payment-double-charge/network-har-timeout-retry.har` — sanitized HAR file capturing the original timed-out request and the retry request (auth tokens and PII redacted before saving, per [Evidence README](../evidence/README.md))
- `evidence/bug-003-payment-double-charge/db-payments-table-duplicate.png` — synthetic screenshot recreation of two payment rows for one booking
- `evidence/bug-003-payment-double-charge/postman-repro-idempotency-check.json` — Postman collection run showing two `POST /payments/confirm` calls without a shared idempotency key both returning `200` with distinct charge IDs

**Severity:** Critical — real customers can be charged twice for a single service; this is a direct financial-trust and potential chargeback/support-burden issue.

**Priority:** P0 — blocks release; payments integrity is explicitly called out as a P0 risk area in the [test plan](../docs/01-test-plan-and-strategy.md).

**Repro Rate:** High, ~80% (8/10) when the artificial timeout window is used to reliably force the timing; without the artificial delay this is harder to hit organically but has been reported anecdotally by support for real users on flaky mobile networks, which is what prompted this investigation.

**Workaround:** Support can currently manually detect and refund duplicate charges when a customer reports it, but there's no proactive detection. I recommended a temporary monitoring query (see [`database-validation/sql-validation-queries.md`](../database-validation/sql-validation-queries.md), "Find duplicate payment records per booking attempt") to catch these before the customer even notices, while the fix is in progress.

**Investigation Notes:**
- Checked the "Pay Now" button in DevTools: it does correctly disable on click (rules out a pure double-click frontend bug — this is specifically the *timeout-then-manual-retry* path, distinct from the double-click case already covered by [TC-PAY-003](../test-cases/05-payments-test-cases.md)).
- Inspected the request payload for both the original and retried request in the Network tab: **no idempotency key or client-generated request ID is included in either request** — this is the root cause. Without a stable key the server has no way to recognize the retry as "the same attempt."
- Cross-checked with the payment provider's own API documentation (public docs, not proprietary) confirming their API supports an `Idempotency-Key` header — the integration simply isn't using it yet.
- Recommended fix to backend dev: generate a UUID client-side at the start of checkout, send it as `Idempotency-Key` on the confirm request, and have the backend persist/check it before processing a charge, so a retried request with the same key returns the original result instead of creating a new charge.
- Also recommended this become a permanent case in the automated regression suite (flagged to SDET) since it depends on precise timing and is easy to silently regress.
