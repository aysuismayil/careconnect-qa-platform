# Evidence Notes — BUG-005 (Checkout price mismatch — frontend/backend/API investigation)

Referenced by [`bug-reports/BUG-005-price-mismatch-frontend-backend.md`](../../bug-reports/BUG-005-price-mismatch-frontend-backend.md).

This bug's evidence spans three layers deliberately, since the investigation itself was about isolating which layer was at fault:

- **`checkout-screen-showing-50.png`** — synthetic recreation of the customer-facing checkout screen frozen at the stale `$50.00` total, captured immediately before the "Pay Now" click.
- **`devtools-network-initial-rate-fetch.png`** — DevTools Network tab capture of the initial `GET /caregivers/{id}` response showing `hourlyRate: 25.00`, and confirming (via the absence of any later request) that the frontend never refreshed this value before submit.
- **`devtools-network-confirm-response.png`** — DevTools capture of the `POST /bookings/confirm` response showing `totalCharged: 80.00` — the moment it became clear the backend and frontend disagreed.
- **`postman-get-booking-verification.json`** — an independent Postman `GET /bookings/{bookingId}` call (outside the browser entirely) confirming `80.00` as the authoritative stored amount, ruling out a browser-side caching artifact.
- **`sql-payment-record.txt`** — SQL query result (see [`database-validation/sql-validation-queries.md`](../../database-validation/sql-validation-queries.md), Query 3) cross-confirming `80.00` at the database layer, the final and most authoritative confirmation of what was actually charged.
