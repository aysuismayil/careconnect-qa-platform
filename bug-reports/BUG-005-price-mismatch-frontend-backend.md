BUG-005 — Customer is charged a different price than what was shown at checkout

**Title/Summary:** [BOOKING] Customer is charged a different price than what was shown at checkout

**Short Description:** The checkout screen shows the caregiver's hourly rate from when the page loaded, and never refreshes it. If the caregiver changes their rate while the customer is still on the checkout screen, the customer ends up charged the new rate — an amount they never saw or agreed to on screen.

**Preconditions:**
- Staging environment.
- Caregiver test account (CG-3010) with an editable hourly rate, starting at $25/hr.
- Customer test account with a valid test payment method.
- Two browser sessions or tabs: one as the caregiver, one as the customer.

**Steps to Reproduce:**
1. As the caregiver (CG-3010), confirm the hourly rate is $25/hr, and leave that session open.
2. As the customer, open CG-3010's profile and start checkout for a 2-hour booking. The checkout summary shows: 2 hr × $25/hr = $50.00.
3. Don't submit payment yet.
4. Switch to the caregiver session and change the rate from $25/hr to $40/hr. Save.
5. Switch back to the customer's checkout screen (don't refresh the page) and click "Pay Now" — the screen still shows $50.00.
6. Check the actual amount charged: via the POST /bookings/confirm response, via Postman (GET /api/v1/bookings/{bookingId}), and via the payment record in the database (see database-validation/sql-validation-queries.md, "Get payment amount for a booking ID").

**Actual Result:**
- The checkout screen never updated — it still showed $50.00 at the moment "Pay Now" was clicked. No request was made to refresh the rate before submitting.
- The booking confirm response returned totalCharged: 80.00 (2 hr × the new $40/hr rate).
- The Postman and database checks both confirmed 80.00 as the actual charged amount — so the customer was charged $80.00 after only ever seeing $50.00 on screen.

**Expected Result:** One of two behaviors, per discussion with PM/design (documented decision, not yet in the code):
1. The price should be locked in at the moment the customer enters checkout, the same way the slot itself is soft-locked (see BUG-002), or
2. If the rate changes mid-checkout, the customer should see a "Price has changed, please review" message with the new total before they can pay.

Either way, silently charging a different amount than what was shown is not acceptable.

**Test Data:**
- Caregiver: CG-3010, rate changed from $25.00/hr → $40.00/hr mid-flow
- Customer: cust-price-qa@example-mail.com
- Booking: 2-hour Elder Care session
- Displayed total at submit time: $50.00
- Actual charged total (confirmed via API and database): $80.00

**Environment/Configuration:**
- Environment: Staging
- Browser: Chrome 126, DevTools Network + Console tabs
- API client: Postman, staging environment collection (see api-testing/postman/)
- Database: read-only staging Postgres access via DBeaver

**Evidence/Attachments:**
- evidence/bug-005-price-mismatch-api/checkout-screen-showing-50.png — checkout screen frozen at $50.00
- evidence/bug-005-price-mismatch-api/devtools-network-initial-rate-fetch.png — the initial hourlyRate: 25.00 response, no refresh call before submit
- evidence/bug-005-price-mismatch-api/devtools-network-confirm-response.png — the confirm response showing totalCharged: 80.00
- evidence/bug-005-price-mismatch-api/postman-get-booking-verification.json — sanitized Postman response confirming 80.00 as the stored amount
- evidence/bug-005-price-mismatch-api/sql-payment-record.txt — sanitized SQL result confirming the 80.00 payment record

**Severity:** High — no data corruption or security exposure, but this is a billing-transparency problem: a customer can be legitimately charged an amount they never saw or approved.

**Priority:** P1 — fix before next release. Not a full P0 only because it requires a caregiver to actively change their rate during another customer's live checkout — a narrower but real scenario.

**Repro Rate:** 5/5 (100%) when the rate change is timed to land between checkout page load and payment submit.

**Workaround:** None for the customer — they have no way to know the price changed underneath them. As an interim step, I suggested temporarily blocking rate edits for caregivers who have an active pending checkout against them, as a smaller/faster fix than full price-locking.

---

**Investigation Notes:** This one needed checking across three layers to pin down where the mismatch came from — frontend, API, and database.
- Frontend (DevTools Console/Network): the checkout screen fetches the caregiver's rate once, on page load, and stores it locally. No polling, no refresh on focus, no refresh triggered by clicking "Pay Now."
- Backend/API (Postman): GET /bookings/{id} returns the true charged amount ($80.00), based on the caregiver's rate at confirmation time. So the backend is using the live rate correctly from its own point of view — this is a frontend/backend contract gap, not a backend calculation bug. The backend assumes the frontend will re-check the price before submit, and it doesn't.
- Database: cross-checked the payments table directly to rule out Postman hitting a cache. The database record matches the API response, so 80.00 really was charged.
- Suggested fix, in order: (1) short-term — re-fetch and show the rate right before enabling "Pay Now," and block submit if it changed since page load; (2) longer-term — extend the existing slot soft-lock to also lock in the price at that moment, so price and availability are held together.
