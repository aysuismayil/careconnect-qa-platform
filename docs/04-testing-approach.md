# Testing Approach — "How I Tested CareConnect"

This document answers the interview-style question "how would you test this application?" — applied directly to CareConnect rather than kept generic. It complements [`01-test-plan-and-strategy.md`](01-test-plan-and-strategy.md) (the *what* and the risk-based *why*) with the concrete *how*: a 5-step approach, and what each step actually looked like on this project.

## Step 1 — Understand the main business goal

CareConnect is a two-sided caregiving marketplace: caregivers list availability and services, customers search and book them, and money changes hands. The business depends on two things above all else: **customers trusting the platform enough to book and pay**, and **bookings actually being reliable** (no double-bookings, no lost payments, no stale search results). That framing drove test prioritization directly — Payments, booking creation/cancellation, and authentication are P0 in the [risk-based prioritization](01-test-plan-and-strategy.md#8-risk-based-prioritization) precisely because a failure there hits revenue and trust at the same time, not because of a generic "auth is always important" rule.

## Step 2 — Requirements / scope review, and clarification questions

Before a story reached dev, it went through a QA review in refinement — see [`02-requirements-and-acceptance-criteria-review.md`](02-requirements-and-acceptance-criteria-review.md) for the full real process and two worked examples. In practice this meant checking every story against a fixed checklist: is each AC testable, are negative paths defined, are boundary values called out, is concurrent/edge timing addressed, is behavior defined across web and mobile. The two examples in that document — the caregiver search radius filter, and booking a time slot — show this wasn't a formality: both stories shipped with real gaps (undefined radius units, no boundary-inclusion rule, no double-booking handling, no payment-failure handling), and both gaps later became real bugs (BUG-001, BUG-002) once the code was tested against the *revised* AC that came out of this review.

## Step 3 — Smoke, functional, happy-path, and E2E testing

Using CareConnect's actual flows (see [`test-cases/`](../test-cases/) for the full structured cases):

- **Smoke check:** after every deploy to staging/production, a critical-path check — can a customer search, view a profile, and reach checkout; can a user sign up, log in, and log out (see [`test-cases/01-authentication-test-cases.md`](../test-cases/01-authentication-test-cases.md)).
- **Functional / happy path:** structured test cases executed against staging before each release, covering sign-up, login, logout & session handling, password reset, and role-based access for both caregiver and customer roles.
- **End-to-end:** the core E2E case for this app is the full booking loop — search for a caregiver within a radius → apply filters → view a profile → select a time slot → complete checkout → receive a booking confirmation. This is the flow every P0 risk area in the Test Plan traces back to, and it's the flow that BUG-002 (double-booking race condition) and BUG-003 (payment double-charge on retry) were found by exercising end-to-end, not by testing checkout in isolation.

## Step 4 — Negative testing and edge cases

This is where most of the real, filed defects in this project came from — six bugs, all traceable to a specific edge case rather than the happy path:

- **Boundary value at the edge of a filter** (a caregiver exactly at the radius limit) → [BUG-001](../bug-reports/BUG-001-search-radius-boundary-excluded.md) — the inclusive-boundary AC from the requirements review wasn't actually implemented that way.
- **Two customers reaching checkout for the same slot at nearly the same time** → [BUG-002](../bug-reports/BUG-002-double-booking-race-condition.md) — a concurrency/race-condition case, not a single-user scenario.
- **Retrying a payment after an ambiguous failure state** → [BUG-003](../bug-reports/BUG-003-payment-double-charge-retry.md).
- **Logging out and checking whether the session is actually invalidated** → [BUG-004](../bug-reports/BUG-004-session-token-not-invalidated-on-logout.md) — a security-adjacent edge case, not just a UI check.
- **Comparing the price shown on the frontend against what the backend actually charges** → [BUG-005](../bug-reports/BUG-005-price-mismatch-frontend-backend.md) — found via database validation, not the UI alone.
- **Losing and regaining network connectivity mid-payment on mobile** → [BUG-006](../bug-reports/BUG-006-mobile-duplicate-charge-on-reconnect.md) — a mobile-specific connectivity edge case.

The pattern: on a marketplace app handling money, the highest-value edge cases are concurrency (two actors, same resource, same moment), state consistency between frontend and backend, and failure/retry handling — not just invalid form input.

## Step 5 — Non-functional testing (only what's relevant)

- **Accessibility** — manual checks (keyboard navigation, screen reader, color contrast) on customer-facing flows; real and evidenced (see `accessibility-testing/`), not a generic checklist item.
- **Mobile-specific behavior** — native app parity for iOS/Android, plus connectivity edge cases like BUG-006; covered in [`test-cases/06-mobile-test-cases.md`](../test-cases/06-mobile-test-cases.md) with a real device/OS matrix.
- **API contract testing** — direct endpoint testing via Postman, independent of the UI (see `api-testing/`).
- **Database validation** — read-only SQL checks confirming backend state matches what the UI shows, which is how BUG-005 was actually found.
- **Performance / load testing** — explicitly out of scope for this QA role (owned by a dedicated performance engineer, per the Test Plan); not claimed here.
- **Automated regression** — owned by the SDET team; I flagged candidate cases for automation but did not build the framework itself (also stated plainly in the Test Plan, not repeated here as if it were my own work).
