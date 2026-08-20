# BUG-006 — Native mobile app creates a duplicate booking/charge after network reconnect mid-checkout

**Title/Summary:**
[Mobile — iOS/Android] On the native app, losing network connectivity mid-payment and having the app auto-retry on reconnect can create a second booking and charge, distinct from the web-specific timeout/retry issue in BUG-003.

**Short Description:**
The native app includes a background auto-retry mechanism for failed network requests (to improve resilience on mobile networks). During checkout, if the "confirm booking" request is in flight when connectivity drops, the app's auto-retry logic resubmits the request once connectivity returns — but it resubmits it as a brand-new request rather than checking whether the original request actually completed server-side first. This can result in two confirmed bookings and two charges from a single checkout attempt.

**Preconditions:**
- iOS or Android native app (staging build) installed on a physical device.
- Customer test account logged in, valid test payment method saved.
- Ability to toggle device airplane mode / network connectivity on demand.
- A caregiver with one open slot available.

**Steps to Reproduce:**
1. On the native app, log in as `cust-mobile-qa@example-mail.com`, select an available slot, and proceed to the final "Pay Now" step.
2. Tap "Pay Now" to submit the booking confirmation request.
3. Immediately (within ~1 second) enable Airplane Mode on the device, simulating a dropped connection right as the request is in flight.
4. Wait 5 seconds, then disable Airplane Mode to restore connectivity.
5. Observe the app's behavior — do not manually tap "Pay Now" again; only observe what the app does automatically on reconnect.
6. Check "My Bookings" in the app for duplicate entries.
7. Cross-verify at the data layer via the SQL query "Find duplicate payment records per booking attempt" (see [`database-validation/sql-validation-queries.md`](../database-validation/sql-validation-queries.md)) and via Postman `GET /api/v1/bookings/me` for the test account.

**Actual Result:**
In 4 of 8 attempts (both iOS and Android, ~50%), the app's auto-retry fired a second `POST /bookings/confirm` request on reconnect without first checking the status of the original request. Two `confirmed` bookings for the same slot/time appeared in "My Bookings," and the SQL/Postman cross-check confirmed two distinct payment records and two charges on the test card.

**Expected Result:**
On reconnect, the app should first check whether the original in-flight request actually completed (e.g., by querying booking status using the same idempotency key/client-generated request ID used server-side — see the related fix recommended in [BUG-003](BUG-003-payment-double-charge-retry.md)) before deciding whether to resubmit. If the original request already succeeded, the app should simply show the existing confirmed booking, not create a new one.

**Test Data:**
- Customer: `cust-mobile-qa@example-mail.com`
- Devices: iPhone 14 (iOS 17), Samsung Galaxy S22 (Android 14)
- Test card: `4242 4242 4242 4242`
- Slot: Elder Care, 2 hours, $50.00

**Environment/Configuration:**
- Environment: Staging
- App build: iOS staging build 4.12.0 (312), Android staging build 4.12.0 (298) (sample version labels)
- Network simulation: device Airplane Mode toggle (manual), also reproduced using network-link conditioner on iOS for a more controlled ~5s drop
- Backend: same `booking-service v1.8.2` / `payments-service v3.1.0` as [BUG-003](BUG-003-payment-double-charge-retry.md)

**Evidence/Attachments:**
- `evidence/bug-006-mobile-duplicate-charge/mobile-app-duplicate-bookings-ios.png` — synthetic recreation of "My Bookings" on iOS showing two entries for the same slot
- `evidence/bug-006-mobile-duplicate-charge/mobile-app-duplicate-bookings-android.png` — synthetic recreation of the same on Android
- `evidence/bug-006-mobile-duplicate-charge/postman-duplicate-bookingids.json` — sanitized Postman export confirming two distinct `bookingId` values with identical `caregiverId`, `startTime`, and `endTime`
- `evidence/bug-006-mobile-duplicate-charge/sql-duplicate-payments.txt` — sanitized SQL result showing two payment records for the one checkout attempt

**Severity:** Critical — same underlying class of issue as BUG-003 (duplicate charge), with an additional mobile-specific trigger (the app's own auto-retry logic) that the web client doesn't have, meaning this needs its own fix even after BUG-003 is resolved.

**Priority:** P0 — blocks release; mobile is a primary usage surface for this product and this compounds an already-critical payments-integrity issue.

**Repro Rate:** ~50% (4/8) across both platforms when connectivity is dropped within 1 second of submitting payment; timing-dependent, consistent with a race between the in-flight request and the reconnect-triggered retry.

**Workaround:** Advise customers experiencing this (if it reaches production) to check "My Bookings" for duplicates after any checkout interrupted by a connectivity drop, and contact support for a refund of the duplicate charge. Not a real workaround, just damage control — flagged as urgent to leadership given the mobile-specific trigger.

**Investigation Notes:**
- This bug shares its root cause category with [BUG-003](BUG-003-payment-double-charge-retry.md) (missing idempotency handling on the confirm endpoint) but has a distinct **trigger mechanism**: BUG-003 is a human manually retrying after a timeout error message; this bug is the **app itself** silently auto-retrying in the background without the user doing anything, which makes it more dangerous — the user may not even know a retry happened.
- Confirmed via the app's debug/console logs (accessible in the staging build) that the mobile app has a generic "retry on reconnect" queue for failed requests, and the booking confirmation call is not excluded from it, unlike some other non-idempotent-sensitive calls in the app which are already correctly excluded (per a code comment I found while pairing with the mobile dev to investigate) — meaning the pattern for fixing this already exists elsewhere in the codebase, it just wasn't applied consistently to this specific endpoint.
- Recommended fix, aligned with BUG-003's recommendation: once the backend supports an `Idempotency-Key` header, the mobile app's retry queue should reuse the same key on any automatic retry of this specific request, and/or the confirm-booking call should be added to the app's existing "do not auto-retry, check status instead" exclusion list.
- Recommended this be retested together with BUG-003's fix rather than independently, since fixing the backend idempotency handling alone should also close this mobile-specific path — flagged this dependency clearly to the team during triage so they aren't fixed and verified in isolation from each other.
