# Evidence Notes — BUG-006 (Mobile duplicate charge on reconnect)

Referenced by [`bug-reports/BUG-006-mobile-duplicate-charge-on-reconnect.md`](../../bug-reports/BUG-006-mobile-duplicate-charge-on-reconnect.md).

- **`mobile-app-duplicate-bookings-ios.png`** — synthetic recreation of the iOS "My Bookings" screen showing two confirmed entries for the same caregiver/time slot after the airplane-mode reconnect repro.
- **`mobile-app-duplicate-bookings-android.png`** — same repro on Android, included to confirm the issue reproduces on both platforms rather than being iOS- or Android-specific.
- **`postman-duplicate-bookingids.json`** — Postman export of `GET /bookings/me` showing two distinct `bookingId` values with identical `caregiverId`, `startTime`, and `endTime`, confirming this at the API layer independent of what either app displayed.
- **`sql-duplicate-payments.txt`** — SQL result (Query 2 in [`database-validation/sql-validation-queries.md`](../../database-validation/sql-validation-queries.md)) confirming two successful payment records tied to the one checkout attempt.
