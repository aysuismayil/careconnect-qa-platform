BUG-002 — Two customers can book the same time slot

**Title/Summary:** [BOOKING] Two customers can book the same time slot

**Short Description:** The booking flow is supposed to hold a slot for 10 minutes once a customer enters checkout, and only reserve it for good once payment goes through. When two customers check out the same slot at almost the same time, both payments can go through and both bookings can end up confirmed for the same time slot. Both customers get charged for a slot only one of them can actually get.

**Preconditions:**
- Staging environment.
- One caregiver profile (CG-2005) with exactly one open slot: Saturday 10:00 AM–12:00 PM.
- Two customer test accounts (cust-a-qa@example-mail.com, cust-b-qa@example-mail.com), each with a valid test payment method.
- Two separate browser sessions, or a browser plus Postman to control the timing more precisely.

**Steps to Reproduce:**
1. As Customer A, go to CG-2005's profile, pick the Saturday 10–12 slot, and get to the final checkout screen. Don't submit yet.
2. As Customer B, in a separate session, get to the same slot's checkout screen. Don't submit yet.
3. Submit payment for both customers within about the same second (coordinated manually, and also reproduced with two near-simultaneous Postman requests to POST /bookings/{slotId}/confirm, to take human click timing out of it).
4. Check the booking status for both customers.
5. Also check the staging database directly (see database-validation/sql-validation-queries.md, "Detect overlapping confirmed bookings for the same caregiver slot") to confirm at the data level, not just what the UI shows.

**Actual Result:** In 3 of 10 timed attempts, both Customer A's and Customer B's bookings ended up confirmed for the same slot, and both test cards were charged. The SQL check found two rows with status = 'confirmed' for the same caregiver_id and overlapping time.

**Expected Result:** Per the reviewed acceptance criteria (see docs/02-requirements-and-acceptance-criteria-review.md, Example 2), only the first customer to complete payment should get a confirmed booking. The second customer should see "This time is no longer available," should not be charged, and the slot should be removed from search.

**Test Data:**
- Caregiver: CG-2005, slot 2026-05-16T10:00:00Z–2026-05-16T12:00:00Z (synthetic future date used in staging)
- Customer A: cust-a-qa@example-mail.com, test card 4242 4242 4242 4242
- Customer B: cust-b-qa@example-mail.com, test card 4000 0566 5566 5556 (different test card, to rule out card-specific caching)
- Requests fired within ≤ 300ms of each other (confirmed via request timestamps in the logs)

**Environment/Configuration:**
- Environment: Staging
- Reproduced both through the UI (two browser sessions) and directly through the API (Postman Runner, two requests back to back), which rules out this being just a frontend button/debounce issue.

**Evidence/Attachments:**
- evidence/bug-002-double-booking/postman-concurrent-requests.json — sanitized Postman run showing both confirm requests and their 200 OK responses
- evidence/bug-002-double-booking/sql-query-result.txt — sanitized SQL result showing two confirmed rows for the same slot
- evidence/bug-002-double-booking/booking-history-both-accounts.png — both accounts showing a confirmed booking for the same time

**Severity:** Critical — real financial impact. Two customers are charged for a service only one can receive, and the caregiver ends up double-booked.

**Priority:** P0 — blocks release. This is a core integrity guarantee of the booking system, also flagged as a P0 risk area in the test plan.

**Repro Rate:** Intermittent, about 30% (3/10 attempts) when requests are fired within a 300ms window. That inconsistency is expected for this type of timing issue.

**Workaround:** None on the customer or support side. As a stopgap, I suggested adding a monitoring alert for "overlapping confirmed bookings for the same caregiver_id" so support can catch and manually fix any that slip through in production until this is fixed.

---

**Investigation Notes:**
- The timing points to both requests checking "is this slot available" before either one finishes writing its booking, and both then going ahead and confirming.
- Checked the slot_holds table during a timed test: a hold row is created for whichever request lands first, but the second request's confirm call doesn't consistently check for an existing hold before charging. This suggests the hold check and the payment confirm step aren't happening as one atomic operation.
- Suggested to the backend dev, as an option, adding a database-level unique constraint (e.g. on caregiver_id + start_time for confirmed bookings) as a hard guarantee, in addition to whatever the application-level check is doing.
- Flagged that this class of bug is easy to miss with manual testing alone and would benefit from a dedicated concurrency/load test in automation.
- This bug is what led to adding an explicit concurrency acceptance criterion during requirements review — the original story didn't have one.
