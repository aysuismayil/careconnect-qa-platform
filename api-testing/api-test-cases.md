# API Test Cases (Postman)

I test the CareConnect API directly with Postman — independent of the UI — to catch issues the frontend might mask (or introduce), validate contract behavior, and cover negative/edge cases faster than clicking through the UI for each one.

The full collection is in [`postman/CareConnect-API.postman_collection.json`](postman/CareConnect-API.postman_collection.json) with a matching environment file at [`postman/CareConnect-Environment.postman_environment.json`](postman/CareConnect-Environment.postman_environment.json). All URLs, tokens, and IDs in both files are placeholders/synthetic — no real endpoints or credentials.

## Why I test at the API level, not just through the UI

- **Speed:** I can run 20 negative-case checks against an endpoint in the time it takes to click through 2 of them in the UI.
- **Isolation:** confirms whether a bug is frontend or backend (see [BUG-005](../bug-reports/BUG-005-price-mismatch-frontend-backend.md), where the Postman check was what proved the backend was charging correctly and the frontend just wasn't re-fetching).
- **Coverage the UI can't easily reach:** malformed payloads, missing headers, invalid IDs, boundary values, and race conditions (see [BUG-002](../bug-reports/BUG-002-double-booking-race-condition.md), reproduced with Postman Runner firing near-simultaneous requests).
- **Regression safety net:** re-running the collection after a bug fix is a fast way to confirm the fix and check for side effects on related endpoints before doing a full UI pass.

## Collection structure

| Folder | Covers |
|---|---|
| `Auth` | Signup, login, logout, token refresh, invalid credentials, token replay after logout |
| `Search` | Search with filters, radius boundary cases, invalid location, malformed query params |
| `Caregiver Profile` | Get profile, update profile (auth required), unauthorized update attempt |
| `Booking` | Create hold, confirm booking, cancel, invalid slot ID, expired hold, double-confirm |
| `Payments` | Confirm payment, idempotency key behavior, tampered price payload, refund |

## Example test cases (a sample from the full collection)

| ID | Request | Scenario | Expected | Priority |
|---|---|---|---|---|
| API-AUTH-01 | `POST /auth/login` | Valid credentials | `200`, returns `accessToken` + `refreshToken` | P0 |
| API-AUTH-02 | `POST /auth/login` | Wrong password | `401`, generic error message, no field-specific hint | P0 |
| API-AUTH-03 | `GET /bookings/me` | Valid token after logout (see [BUG-004](../bug-reports/BUG-004-session-token-not-invalidated-on-logout.md)) | Should be `401`; currently returns `200` (bug) | P1 |
| API-SRCH-01 | `GET /search?lat=..&lng=..&radiusMiles=10` | Caregiver at exactly 10.00 mi (see [BUG-001](../bug-reports/BUG-001-search-radius-boundary-excluded.md)) | Should include the caregiver; currently excludes it (bug) | P1 |
| API-SRCH-02 | `GET /search?lat=..&lng=..&radiusMiles=-5` | Negative radius | `400 Bad Request` with validation message, not a server error | P2 |
| API-BOOK-01 | `POST /bookings/{slotId}/hold` | Valid open slot | `200`, hold created with `expiresAt` ~10 min out | P0 |
| API-BOOK-02 | `POST /bookings/{slotId}/confirm` | Two near-simultaneous confirms on the same slot (see [BUG-002](../bug-reports/BUG-002-double-booking-race-condition.md)) | Only one should succeed; currently both can succeed intermittently (bug) | P0 |
| API-BOOK-03 | `POST /bookings/{slotId}/confirm` | Slot ID that doesn't exist | `404 Not Found`, no stack trace leaked in response body | P1 |
| API-PAY-01 | `POST /payments/confirm` | Retried request without idempotency key (see [BUG-003](../bug-reports/BUG-003-payment-double-charge-retry.md)) | Should be treated as idempotent; currently creates a duplicate charge (bug) | P0 |
| API-PAY-02 | `POST /payments/confirm` | Tampered `amount` field lower than server-calculated price | Server should recalculate server-side and ignore/reject the client value | P0 |
| API-PAY-03 | `POST /payments/refund` | Refund a booking cancelled outside the refund window | `200`, correct partial-refund amount per policy, itemized in response | P1 |

## How I structure Postman tests

Every request in the collection has a **Tests** script that asserts on status code, response shape, and key field values — not just "it returned something." Example pattern used throughout the collection:

```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Response has expected booking fields", function () {
    const body = pm.response.json();
    pm.expect(body).to.have.property("bookingId");
    pm.expect(body).to.have.property("status");
    pm.expect(body.status).to.be.oneOf(["pending_caregiver_response", "confirmed"]);
});

pm.test("Total charged matches expected calculation", function () {
    const body = pm.response.json();
    pm.expect(body.totalCharged).to.eql(pm.environment.get("expectedTotal"));
});
```

I also use a **pre-request script** on the environment to fetch and store a fresh auth token automatically before each run, so the whole collection (or a subset via Collection Runner) can be executed unattended for quick regression passes after a deploy.
