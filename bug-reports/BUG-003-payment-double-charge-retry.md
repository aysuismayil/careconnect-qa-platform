BUG-003 — Customer is charged twice after retrying a failed payment

**Title/Summary:** [PAYMENT] Customer is charged twice after retrying a failed payment

**Short Description:** When the payment request times out on the customer's screen but actually went through on the server, the customer sees an error and naturally tries again. That retry creates a second, separate charge for the same booking instead of recognizing the first one already succeeded. Customers can end up charged twice for one booking.

**Preconditions:**
- Staging environment with network throttling/interception available (Chrome DevTools Network tab, or a proxy tool).
- Customer test account with a valid test payment method.
- A caregiver with an open slot available.

**Steps to Reproduce:**
1. Log in as a customer test account, pick an available slot, and go to checkout.
2. Simulate a dropped connection: the request reaches the server and succeeds, but the client never gets the response (done in staging with an artificial server-side delay plus killing the client connection mid-response, coordinated with a backend dev for this specific test).
3. Submit payment.
4. The client times out and shows a generic error ("Something went wrong, please try again").
5. Click "Pay Now" again with the same payment details.
6. Check the customer's booking history and the number of payment records tied to this booking attempt.

**Actual Result:** Two payment records are created, and two separate charges go through on the same test card for what the customer experienced as one booking attempt. The confirmed booking only appears after the second charge — the first charge (which actually succeeded) sits with no booking cleanly linked to it.

**Expected Result:** The retry should be recognized as the same booking attempt (using an idempotency key generated at the start of checkout and reused on retry) and shouldn't create a second charge. Either the original successful payment/booking should just be returned to the client, or if a new attempt really is needed, the first charge should be reliably voided first.

**Test Data:**
- Customer: cust-retry-qa@example-mail.com
- Test card: 4242 4242 4242 4242, exp 12/29, CVC 123
- Booking attempt: Elder Care, 2 hours, $60.00 total
- Simulated delay: artificial 8-second server-side hold on the confirm endpoint combined with a 5-second client timeout, to reliably reproduce a "client gave up, server kept processing" scenario

**Environment/Configuration:**
- Environment: Staging
- Browser: Chrome 126, DevTools network throttling ("Slow 3G" plus a custom timeout override)
- Payment provider: sandbox/test mode only

**Evidence/Attachments:**
- evidence/bug-003-payment-double-charge/network-har-timeout-retry.har — sanitized HAR file with the original timed-out request and the retry (auth tokens and PII redacted)
- evidence/bug-003-payment-double-charge/db-payments-table-duplicate.png — two payment rows for one booking
- evidence/bug-003-payment-double-charge/postman-repro-idempotency-check.json — Postman run showing two POST /payments/confirm calls without a shared idempotency key, both returning 200 with different charge IDs

**Severity:** Critical — real customers can be charged twice for a single service. Direct financial-trust and support-burden issue.

**Priority:** P0 — blocks release. Payments integrity is called out as a P0 risk area in the test plan.

**Repro Rate:** High, ~80% (8/10) when using the artificial timeout to force the timing. Without the artificial delay it's harder to hit on purpose, but has come up anecdotally from support for real users on flaky mobile networks — that's actually what prompted this investigation.

**Workaround:** Support can manually detect and refund duplicate charges when a customer reports one, but there's no proactive detection yet. I suggested a temporary monitoring query (see database-validation/sql-validation-queries.md, "Find duplicate payment records per booking attempt") to catch these before the customer notices, while the fix is in progress.

---

**Investigation Notes:**
- Checked the "Pay Now" button in DevTools: it does correctly disable on click, which rules out a plain double-click bug (that's a separate case already covered by TC-PAY-003). This is specifically the timeout-then-manual-retry path.
- Checked the request payload for both the original and retried request in the Network tab: neither one includes an idempotency key or client-generated request ID. Without a stable key, the server has no way to tell the retry is "the same attempt" as the first one.
- Confirmed in the payment provider's public API docs that their API supports an Idempotency-Key header — it just isn't being used in this integration yet.
- Suggested to the backend dev: generate a key client-side at the start of checkout, send it as Idempotency-Key on the confirm request, and have the backend check it before processing a charge, so a retry with the same key returns the original result instead of creating a new charge.
- Flagged this as a good candidate for a permanent regression test case, since it depends on precise timing and could regress silently.
