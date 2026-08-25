BUG-006 — Duplicate booking and charge are created after connection is restored during checkout

**Title/Summary:** [MOBILE] Duplicate booking and charge are created after connection is restored during checkout

**Short Description:** The native app has a background auto-retry for failed network requests, meant to help on flaky mobile connections. If the "confirm booking" request is in flight when the connection drops, the app automatically resubmits it once the connection is back — but it resubmits it as a brand-new request without checking if the original one already went through. This can create two confirmed bookings and two charges from one checkout.

**Preconditions:**
- iOS or Android native app (staging build) on a physical device.
- Customer test account logged in, with a saved test payment method.
- Ability to toggle airplane mode on the device.
- A caregiver with one open slot available.

**Steps to Reproduce:**
1. On the native app, log in as cust-mobile-qa@example-mail.com, pick an available slot, and go to the final "Pay Now" step.
2. Tap "Pay Now."
3. Within about 1 second, turn on Airplane Mode to simulate the connection dropping mid-request.
4. Wait 5 seconds, then turn Airplane Mode off.
5. Don't tap "Pay Now" again — just watch what the app does on its own after reconnecting.
6. Check "My Bookings" for duplicate entries.
7. Also check the SQL query "Find duplicate payment records per booking attempt" (see database-validation/sql-validation-queries.md) and Postman GET /api/v1/bookings/me for the test account.

**Actual Result:** In 4 of 8 attempts (both iOS and Android, about 50%), the app auto-retried the confirm request on reconnect without checking if the original request had already gone through. Two confirmed bookings for the same slot showed up in "My Bookings," and the SQL/Postman check confirmed two separate payment records and two charges.

**Expected Result:** On reconnect, the app should first check whether the original request actually completed (e.g. by checking booking status using the same idempotency key/request ID mentioned in BUG-003) before deciding to resubmit. If the original request already succeeded, the app should just show the existing confirmed booking instead of creating a new one.

**Test Data:**
- Customer: cust-mobile-qa@example-mail.com
- Devices: iPhone 14 (iOS 17), Samsung Galaxy S22 (Android 14)
- Test card: 4242 4242 4242 4242
- Slot: Elder Care, 2 hours, $50.00

**Environment/Configuration:**
- Environment: Staging
- Network simulation: device Airplane Mode toggle (manual), also reproduced with a network-link conditioner on iOS for a more controlled ~5 second drop

**Evidence/Attachments:**
- evidence/bug-006-mobile-duplicate-charge/mobile-app-duplicate-bookings-ios.png — "My Bookings" on iOS showing two entries for the same slot
- evidence/bug-006-mobile-duplicate-charge/mobile-app-duplicate-bookings-android.png — same on Android
- evidence/bug-006-mobile-duplicate-charge/postman-duplicate-bookingids.json — sanitized Postman export confirming two distinct booking IDs with the same caregiver and time
- evidence/bug-006-mobile-duplicate-charge/sql-duplicate-payments.txt — sanitized SQL result showing two payment records for one checkout attempt

**Severity:** Critical — same underlying issue as BUG-003 (duplicate charge), plus a mobile-specific trigger (the app's own auto-retry) that the web client doesn't have. Needs its own fix even after BUG-003 is resolved.

**Priority:** P0 — blocks release. Mobile is a primary usage surface for this product, and this compounds an already-critical payments issue.

**Repro Rate:** ~50% (4/8) across both platforms, when connectivity is dropped within 1 second of submitting payment. Timing-dependent, which is expected for this kind of issue.

**Workaround:** If this reaches production, customers should check "My Bookings" for duplicates after any checkout interrupted by a connection drop, and contact support for a refund of the extra charge. Not a real fix, just damage control.

---

**Investigation Notes:**
- This shares its root cause with BUG-003 (no idempotency handling on the confirm endpoint), but the trigger is different: BUG-003 is a person manually retrying after seeing an error; this is the app retrying on its own in the background, without the user doing anything — which is arguably worse, since the user may never know a retry happened.
- Checked the app's debug logs in the staging build: there's a generic "retry on reconnect" queue for failed requests, and the booking confirmation call is included in it.
- Recommended fix, in line with BUG-003: once the backend supports an Idempotency-Key header, the app's retry queue should reuse the same key on any automatic retry, and/or the confirm-booking call should be excluded from auto-retry and instead check status on reconnect.
- Recommended this be retested together with BUG-003's fix rather than separately, since fixing the backend idempotency handling should also close this path — flagged that dependency to the team during triage.
