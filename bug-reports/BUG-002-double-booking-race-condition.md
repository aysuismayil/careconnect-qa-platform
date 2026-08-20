# BUG-002 — Race condition allows two customers to both confirm the same booking slot

**Title/Summary:**
[Booking] Concurrent checkout by two customers on the same caregiver time slot can result in both bookings being confirmed and charged, instead of the second customer being blocked as designed.

**Short Description:**
The booking flow is supposed to "soft-lock" a slot for 10 minutes once a customer enters checkout, and permanently reserve it only after payment succeeds. Under near-simultaneous checkout by two different customers for the same slot, both payments can succeed and both bookings can reach `confirmed` status for the same caregiver time slot — a direct double-booking with two customers charged for a slot only one of them can actually receive.

**Preconditions:**
- Staging environment.
- One caregiver profile (`CG-2005`) with exactly one open, unbooked slot: Saturday 10:00 AM–12:00 PM.
- Two distinct customer test accounts (`cust-a-qa@example-mail.com`, `cust-b-qa@example-mail.com`), each with a valid test payment method.
- Two separate browser sessions (or one browser + Postman collection runner to control timing precisely).

**Steps to Reproduce:**
1. As Customer A, navigate to `CG-2005`'s profile and select the Saturday 10–12 slot; proceed to the final checkout/payment screen but do not submit yet.
2. As Customer B, in a separate session, navigate to the same slot and proceed to the same final checkout/payment screen, also without submitting yet.
3. Submit payment for Customer A and Customer B within the same ~1-second window (coordinated manually, and separately reproduced via two near-simultaneous Postman requests hitting the `POST /bookings/{slotId}/confirm` endpoint to remove human click-timing variance).
4. Check the booking status for both customers.
5. Query the staging database directly (see [`database-validation/sql-validation-queries.md`](../database-validation/sql-validation-queries.md), query "Detect overlapping confirmed bookings for the same caregiver slot") to confirm at the data layer, not just the UI.

**Actual Result:**
In 3 of 10 timed attempts, both Customer A's and Customer B's bookings reached `confirmed` status for the identical slot, and both customers' test cards were successfully charged. The SQL check confirmed two `bookings` rows with `status = 'confirmed'` referencing the same `caregiver_id` and overlapping `start_time`/`end_time`.

**Expected Result:**
Per the reviewed acceptance criteria (see [`docs/02-requirements-and-acceptance-criteria-review.md`](../docs/02-requirements-and-acceptance-criteria-review.md), Example 2), only the first customer to complete payment should have their booking confirmed. The second customer should see "This time is no longer available," should not be charged, and should be returned to search with the slot removed from results.

**Test Data:**
- Caregiver: `CG-2005`, slot `2026-05-16T10:00:00Z`–`2026-05-16T12:00:00Z` (synthetic future date used in staging)
- Customer A: `cust-a-qa@example-mail.com`, test card `4242 4242 4242 4242`
- Customer B: `cust-b-qa@example-mail.com`, test card `4000 0566 5566 5556` (a different valid test card, to rule out card-specific caching)
- Requests fired within ≤ 300ms of each other, confirmed via request timestamps in the API logs

**Environment/Configuration:**
- Environment: Staging
- Build: `booking-service v1.8.2` (sample version label)
- API base: staging booking API (internal, not documented here)
- Reproduced both via UI (two browser sessions) and directly via API (Postman Runner firing two requests back-to-back) — API-level repro isolates this from any frontend debounce/disable-button behavior

**Evidence/Attachments:**
- `evidence/bug-002-double-booking/postman-concurrent-requests.json` — sanitized Postman run export showing both `confirm` requests and their 200 OK responses
- `evidence/bug-002-double-booking/sql-query-result.txt` — sanitized output of the overlapping-bookings SQL query showing two confirmed rows for the same slot
- `evidence/bug-002-double-booking/booking-history-both-accounts.png` — synthetic screenshot recreation showing both customer accounts with a "confirmed" booking for the same time

**Severity:** Critical — this is a direct business-logic failure with real financial and trust impact: two customers are charged for a service only one can receive, and the caregiver has a scheduling conflict they didn't create.

**Priority:** P0 — blocks release; this is a core integrity guarantee of the booking system (also flagged as a P0 risk area in the [test plan](../docs/01-test-plan-and-strategy.md)).

**Repro Rate:** Intermittent, approximately 30% (3/10 attempts) when requests are fired within a 300ms window — consistent with a race condition rather than a deterministic bug, which is itself informative for the dev investigating it.

**Workaround:** None identified from the customer or support side. As a stopgap, I recommended the team add a monitoring alert on "overlapping confirmed bookings for the same caregiver_id" so support can proactively catch and manually resolve any that slip through in production until this is fixed.

**Investigation Notes:**
- Timing strongly suggests a classic check-then-act race: both requests likely read "slot is available" before either write completes, then both proceed to write a `confirmed` booking.
- Checked whether the 10-minute soft-lock was even being applied at the DB level: queried the `slot_holds` table during a timed test and found a hold row was created for whichever request landed first, but the second request's confirm call did not consistently check for an existing hold before proceeding to charge — suggesting the hold-check and the payment-confirm step are not wrapped in a single atomic transaction/lock.
- Recommended to the backend dev: use a unique constraint (e.g., a partial unique index on `caregiver_id + start_time` for `status IN ('confirmed')`) as a hard database-level guarantee, rather than relying solely on application-level hold logic — defense in depth, since the current app-level check is clearly racy under load.
- Also flagged that this should be covered by a load/concurrency test in the automated suite going forward (raised with the SDET), since manual timing-based repro is inherently inconsistent and this class of bug is easy to miss without dedicated concurrency testing.
- This bug directly informed the AC change captured in requirements review — the original story had no explicit concurrency AC at all.
