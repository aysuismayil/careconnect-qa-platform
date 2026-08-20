# Database / SQL Validation

As a manual QA engineer, I had **read-only** access to a staging PostgreSQL database, which I used to confirm that actions taken through the UI or API actually produced the correct backend state — not just a correct-looking screen. This catches bugs where the frontend shows success but the data underneath is wrong (or vice versa).

All queries below are written against a simplified, sanitized schema that mirrors the real one closely enough to be representative, without exposing actual table/column names or any real data.

> **Access model:** I never had write access to staging or production databases. All queries here are `SELECT` only — I verify state, I don't change it.

---

## Simplified schema used in these examples

```sql
-- users: both customers and caregivers
users (id, email, role, created_at)

-- caregiver_profiles: caregiver-specific profile data
caregiver_profiles (id, user_id, hourly_rate, is_active, created_at, updated_at)

-- bookings: one row per booking attempt
bookings (
  id, customer_id, caregiver_id, slot_start, slot_end,
  status,            -- 'held' | 'pending_caregiver_response' | 'confirmed' | 'declined' | 'cancelled_by_customer' | 'completed'
  created_at, updated_at
)

-- payments: one row per charge attempt
payments (
  id, booking_id, amount, status, idempotency_key, created_at
)

-- slot_holds: temporary holds during checkout
slot_holds (id, booking_id, slot_id, expires_at, created_at)
```

---

## Query 1 — Detect overlapping confirmed bookings for the same caregiver slot

Used to confirm [BUG-002](../bug-reports/BUG-002-double-booking-race-condition.md) at the data layer, not just via API responses.

```sql
SELECT
    b1.id  AS booking_id_1,
    b2.id  AS booking_id_2,
    b1.caregiver_id,
    b1.slot_start,
    b1.slot_end
FROM bookings b1
JOIN bookings b2
    ON b1.caregiver_id = b2.caregiver_id
    AND b1.id < b2.id
    AND b1.status = 'confirmed'
    AND b2.status = 'confirmed'
    AND b1.slot_start < b2.slot_end
    AND b2.slot_start < b1.slot_end
ORDER BY b1.slot_start;
```

**What I'm checking:** this should always return **zero rows** in a healthy system. Any row returned means two confirmed bookings overlap for the same caregiver — a double-booking.

---

## Query 2 — Find duplicate payment records per booking attempt

Used to confirm [BUG-003](../bug-reports/BUG-003-payment-double-charge-retry.md) and [BUG-006](../bug-reports/BUG-006-mobile-duplicate-charge-on-reconnect.md).

```sql
SELECT
    booking_id,
    COUNT(*)          AS payment_count,
    SUM(amount)        AS total_charged,
    ARRAY_AGG(id)       AS payment_ids,
    ARRAY_AGG(idempotency_key) AS idempotency_keys
FROM payments
WHERE status = 'succeeded'
GROUP BY booking_id
HAVING COUNT(*) > 1
ORDER BY total_charged DESC;
```

**What I'm checking:** any `booking_id` with more than one successful payment is a duplicate charge. I also inspect `idempotency_keys` in the result — distinct/null keys per duplicate confirm the root cause (missing or unused idempotency key).

---

## Query 3 — Get payment amount for a booking ID (cross-check against UI/API)

Used in [BUG-005](../bug-reports/BUG-005-price-mismatch-frontend-backend.md) to independently confirm the actual charged amount at the source of truth.

```sql
SELECT
    p.id            AS payment_id,
    p.booking_id,
    p.amount        AS amount_charged,
    b.slot_start,
    b.slot_end,
    cp.hourly_rate  AS caregiver_rate_at_query_time
FROM payments p
JOIN bookings b            ON b.id = p.booking_id
JOIN caregiver_profiles cp ON cp.id = b.caregiver_id
WHERE p.booking_id = :booking_id;
```

**What I'm checking:** `amount_charged` against what was displayed in the UI at checkout time (captured via DevTools/screenshot) and against what the API's `GET /bookings/{id}` response claims — all three should agree.

---

## Query 4 — Confirm a cancelled booking's refund matches policy

```sql
SELECT
    b.id AS booking_id,
    b.status,
    p.amount AS original_amount,
    r.amount AS refund_amount,
    ROUND((r.amount / NULLIF(p.amount, 0)) * 100, 1) AS refund_percentage,
    b.updated_at AS cancelled_at,
    b.slot_start
FROM bookings b
JOIN payments p ON p.booking_id = b.id AND p.status = 'succeeded'
LEFT JOIN refunds r ON r.payment_id = p.id
WHERE b.status = 'cancelled_by_customer'
ORDER BY b.updated_at DESC
LIMIT 20;
```

**What I'm checking:** for cancellations made ≥24h before `slot_start`, `refund_percentage` should be 100%. For cancellations made <24h before, it should match the documented late-cancellation fee percentage (e.g., 50%). Any row that doesn't match either expected value gets flagged for [TC-PAY-006/TC-PAY-007](../test-cases/05-payments-test-cases.md) follow-up.

---

## Query 5 — Sanity-check search radius filtering directly against stored coordinates

Used alongside [BUG-001](../bug-reports/BUG-001-search-radius-boundary-excluded.md) to compute "ground truth" distance independent of the application's own search service, so I'm not just trusting the same code path that has the bug.

```sql
SELECT
    cp.id AS caregiver_id,
    cp.is_active,
    ROUND(
      3958.8 * ACOS(
        COS(RADIANS(:origin_lat)) * COS(RADIANS(cp.lat)) *
        COS(RADIANS(cp.lng) - RADIANS(:origin_lng)) +
        SIN(RADIANS(:origin_lat)) * SIN(RADIANS(cp.lat))
      ), 2
    ) AS distance_miles
FROM caregiver_profiles cp
WHERE cp.is_active = true
ORDER BY distance_miles ASC;
```

**What I'm checking:** this haversine calculation is my independent "ground truth" for distance, run directly against seeded coordinates. Comparing this against what the `/search` API actually returns for a given radius is how I confirmed the boundary-exclusion bug was real and not a data-seeding mistake on my part.

---

## Why this matters for QA

Testing only through the UI means trusting that the UI, API, and database all agree — but bugs often live exactly in the gaps between those layers (see BUG-005, where the UI and API/DB genuinely disagreed). Writing my own independent SQL checks means I'm not just re-verifying the same assumption the application already makes; I'm checking the actual source of truth.
