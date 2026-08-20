# Evidence Notes — BUG-002 (Double-booking race condition)

Referenced by [`bug-reports/BUG-002-double-booking-race-condition.md`](../../bug-reports/BUG-002-double-booking-race-condition.md).

- **`postman-concurrent-requests.json`** — exported Postman Collection Runner results from firing two `POST /bookings/{slotId}/confirm` requests within a 300ms window, showing both returning `200 OK` with distinct `bookingId` values for the same slot — the core proof of the race condition, captured independently of the UI to rule out a frontend timing artifact.
- **`sql-query-result.txt`** — output of the "Detect overlapping confirmed bookings" SQL query (see [`database-validation/sql-validation-queries.md`](../../database-validation/sql-validation-queries.md), Query 1) run immediately after the Postman repro, confirming two `confirmed` rows exist at the database layer for the same caregiver/slot — this is what elevated the finding from "the API returned something odd" to "the database itself has an integrity violation."
- **`booking-history-both-accounts.png`** — synthetic recreation of both customer test accounts' "My Bookings" screens side by side, each showing a confirmed booking for the identical time slot.
