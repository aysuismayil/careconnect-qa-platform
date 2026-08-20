# BUG-005 — Displayed checkout price doesn't match the amount actually charged (stale price after caregiver rate change)

**Title/Summary:**
[Booking/Payments — Frontend/Backend/API] Checkout screen displays a stale, cached hourly rate when a caregiver updates their price mid-flow, but the backend charges the *new* rate — customer is charged a different amount than what they saw and approved on screen.

**Short Description:**
This bug required investigating across three layers — browser DevTools (frontend), the checkout API response (via DevTools Network tab), and direct API calls (via Postman) — to pin down where the mismatch originated. The customer-facing checkout screen shows the caregiver's hourly rate at the time the page loaded and does not refresh it, but the backend `POST /bookings/confirm` endpoint recalculates price from the caregiver's *current* rate at confirmation time. If the caregiver changes their rate between when the customer opens checkout and when they confirm payment, the customer is charged an amount that never matched anything displayed on screen — a transparency and trust problem, and arguably a billing-accuracy issue.

**Preconditions:**
- Staging environment.
- Caregiver test account (`CG-3010`) with an editable hourly rate, starting at $25/hr.
- Customer test account with a valid test payment method.
- Two browser sessions or tabs: one as the caregiver (to change the rate), one as the customer (to run checkout).
- Chrome DevTools open on the customer session (Network + Console tabs).
- Postman available to independently query the booking price after the fact.

**Steps to Reproduce:**
1. As the caregiver (`CG-3010`), confirm hourly rate is $25/hr; leave that session open.
2. As the customer, open `CG-3010`'s profile and start checkout for a 2-hour booking. The checkout summary displays: `2 hr × $25/hr = $50.00`.
3. Do **not** submit payment yet. Open DevTools → Network tab on the customer session and locate the initial page-load API call that returned the caregiver's rate (e.g., `GET /caregivers/CG-3010`), confirming the response payload shows `hourlyRate: 25.00` — this is the value the UI is displaying and holding in local state.
4. Switch to the caregiver session and change the hourly rate from $25/hr to $40/hr; save.
5. Switch back to the customer's still-open checkout screen (do **not** refresh the page) and click "Pay Now" using the still-displayed "$50.00" total.
6. In DevTools → Network tab, inspect the `POST /bookings/confirm` request and response.
7. Independently confirm the actual charged amount by querying the booking via Postman: `GET /api/v1/bookings/{bookingId}` using a valid staging API token.
8. Cross-check the actual charge amount against the payment record directly in the database (see [`database-validation/sql-validation-queries.md`](../database-validation/sql-validation-queries.md), query "Get payment amount for a booking ID").

**Actual Result:**
- The checkout screen never updated and still showed `$50.00` at the moment "Pay Now" was clicked (confirmed via DevTools — no re-fetch of the caregiver's rate occurred between page load and submit; confirmed in the Console/Network tab that no request was made to refresh `hourlyRate` before submission).
- The `POST /bookings/confirm` response returned `totalCharged: 80.00` (2 hr × the *new* $40/hr rate) — the backend correctly used the live rate, but the customer was never shown this before being charged.
- The Postman query against `GET /api/v1/bookings/{bookingId}` confirmed `80.00` as the stored, authoritative charged amount, matching the database payment record — ruling out a display-only glitch and confirming the customer was actually charged $80.00 after having only ever seen $50.00.

**Expected Result:**
One of two acceptable behaviors, per discussion with PM/design (documented decision, not yet reflected in code):
1. The price should be locked at the moment the customer enters checkout (matching the same 10-minute soft-lock already used for slot availability — see [BUG-002](BUG-002-double-booking-race-condition.md) and the related [requirements review](../docs/02-requirements-and-acceptance-criteria-review.md)), OR
2. If the rate changes mid-checkout, the customer must see an explicit "Price has changed, please review" interstitial with the new total before they can submit payment.
Silently charging a different amount than what was displayed is not acceptable under either option.

**Test Data:**
- Caregiver: `CG-3010`, rate changed from `$25.00/hr` → `$40.00/hr` mid-flow
- Customer: `cust-price-qa@example-mail.com`
- Booking: 2-hour Elder Care session
- Displayed total at submit time: `$50.00`
- Actual charged total (confirmed via API + DB): `$80.00`

**Environment/Configuration:**
- Environment: Staging
- Build: `web-client v4.12.0`, `booking-service v1.8.2` (sample version labels)
- Browser: Chrome 126, DevTools Network + Console tabs
- API client: Postman v10, staging environment collection (see [`api-testing/postman/`](../api-testing/postman/))
- Database: read-only staging Postgres access via DBeaver

**Evidence/Attachments:**
- `evidence/bug-005-price-mismatch-api/checkout-screen-showing-50.png` — synthetic recreation of the checkout screen frozen at $50.00
- `evidence/bug-005-price-mismatch-api/devtools-network-initial-rate-fetch.png` — DevTools capture showing the initial `hourlyRate: 25.00` response and no subsequent refresh call before submit
- `evidence/bug-005-price-mismatch-api/devtools-network-confirm-response.png` — DevTools capture of the `POST /bookings/confirm` response showing `totalCharged: 80.00`
- `evidence/bug-005-price-mismatch-api/postman-get-booking-verification.json` — sanitized Postman response body confirming `80.00` as the authoritative stored amount
- `evidence/bug-005-price-mismatch-api/sql-payment-record.txt` — sanitized SQL query result cross-confirming the `80.00` payment record at the database layer

**Severity:** High — no data corruption or security exposure, but this is a direct billing-transparency problem: a customer can be legitimately charged an amount they never saw or approved, which is a serious trust and potential compliance issue (billing must reflect what the customer agreed to).

**Priority:** P1 — fix before next release; not a full release-blocking P0 only because the reproduction requires a caregiver to actively change their rate during another customer's live checkout window, which is a narrower (but real and non-negligible) scenario.

**Repro Rate:** 5/5 (100%) when the rate change is timed to land between checkout page-load and payment submission.

**Workaround:** None for the customer — they have no way to know the price changed underneath them. As an interim mitigation, I suggested the team consider temporarily disabling rate edits for caregivers who have an active pending checkout against them (a smaller, faster fix than full price-locking), while option 1 or 2 above is implemented properly.

**Investigation Notes:**
This is the clearest example from this project of tracing a bug across the full stack rather than just reporting what the UI showed:
- **Frontend (DevTools Console/Network):** Confirmed the checkout component fetches the caregiver's rate exactly once, on initial page load, and stores it in local component state with no polling, no re-fetch on focus, and no re-fetch triggered by the "Pay Now" click handler. This rules out "the UI is technically re-checking but displaying wrong" — it's simply never re-checking at all.
- **Backend/API (Postman):** Confirmed independently, outside the browser entirely, that `GET /bookings/{id}` returns the true charged amount ($80.00) sourced from the caregiver's rate *at confirmation time*, proving the backend is (correctly, from its own perspective) treating the caregiver's live rate as the source of truth — it's not a backend calculation bug, it's a **frontend/backend contract gap**: the backend assumes the frontend will re-validate price before submit, and the frontend never does.
- **Database:** Cross-checked the `payments` table directly to remove any possibility that Postman itself was hitting a cache — the database record matches the API response, confirming `80.00` really was charged, not just displayed inconsistently somewhere in a cache layer.
- Recommended fix path to the team, in priority order: (1) short-term — re-fetch and re-display the rate immediately before enabling the "Pay Now" button, blocking submission if it changed since page load; (2) proper fix — extend the existing slot soft-lock mechanism to also lock in price at that moment, so price and availability are held together as one atomic reservation.
- This bug is a good example of why I don't stop at "what does the screen show" — the API and database checks were what actually proved *whether the customer's money was affected*, which is the real business risk here, not just a cosmetic display bug.
