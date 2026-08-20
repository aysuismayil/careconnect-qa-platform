# Requirements & Acceptance Criteria Review

One of my core responsibilities was reviewing user stories **before** development started — during backlog refinement/grooming — to catch missing edge cases, ambiguous wording, and untestable acceptance criteria early, when they're cheap to fix.

This document shows my actual review process with two real examples (sanitized) of stories I reviewed, the gaps I found, and how the acceptance criteria changed as a result.

---

## My review checklist

When a story came into refinement, I checked it against this list before giving it a "ready for dev" QA thumbs-up:

- [ ] Is each acceptance criterion phrased as a testable, pass/fail statement (not "should work well")?
- [ ] Are negative/error paths defined, not just the happy path?
- [ ] Are boundary values called out (min/max length, zero results, exact limits)?
- [ ] Is expected behavior defined for concurrent/edge timing issues (e.g., two users booking the same slot)?
- [ ] Are validation rules explicit (required fields, formats, character limits)?
- [ ] Is behavior defined across web and mobile, or does it need separate criteria?
- [ ] Are accessibility expectations included for new UI?
- [ ] Does the story define what happens on network failure / API error / timeout?
- [ ] Is there a clear definition of "done" that QA can verify without guessing?

---

## Example 1 — Caregiver search radius filter

**Original story (as written):**

> As a customer, I want to search for caregivers near me so that I can find someone convenient to book.
>
> **AC:**
> - User can enter a location and see nearby caregivers.
> - User can filter by distance.

**Gaps I raised in refinement:**

1. "Nearby" and "distance" aren't defined — what unit (miles/km)? What's the default radius if the user doesn't pick one?
2. No AC for **zero results** — what does the customer see if no caregivers match?
3. No AC for **invalid location input** (unrecognized address, empty field, user denies location permission).
4. No AC for what happens when a caregiver is exactly on the radius boundary (e.g., exactly 10.0 miles when the filter is "within 10 miles") — inclusive or exclusive?
5. No AC for how radius filtering interacts with other filters (price, rating) applied at the same time.
6. No mention of whether the radius is calculated as straight-line distance or driving distance — this materially changes what caregivers appear.

**Revised AC after my review (what shipped):**

- Default search radius is 15 miles from the entered location if the user doesn't set one.
- Distance is straight-line (haversine) distance in miles, displayed on each caregiver card (e.g., "3.2 mi away").
- Radius filter is **inclusive** of the boundary value (a caregiver at exactly 10.0 mi appears when filter = "within 10 miles").
- If location can't be resolved (invalid/empty input), show inline validation error: "We couldn't find that location. Try a city, ZIP code, or address."
- If zero caregivers match the combined filters, show an empty state with a suggestion to widen the radius or clear filters — never a blank page or infinite spinner.
- Radius filter combines with all other active filters using AND logic.
- This became the basis for [BUG-001](../bug-reports/BUG-001-search-radius-boundary-excluded.md), where I later found the *inclusive boundary* AC was not actually implemented correctly.

---

## Example 2 — Booking a caregiver time slot

**Original story (as written):**

> As a customer, I want to book a caregiver for a specific date and time so that I can schedule care.
>
> **AC:**
> - User selects a date/time and confirms the booking.
> - Caregiver receives a notification.

**Gaps I raised in refinement:**

1. No AC for **double-booking**: what happens if two customers try to book the same caregiver slot at nearly the same time?
2. No AC for what "confirms the booking" actually locks in — is the slot held during checkout, or only after payment succeeds?
3. No AC for booking **cancellation** windows or fees.
4. No AC for caregiver-side **decline/no-response** handling — is the booking auto-confirmed, or does the caregiver need to accept it?
5. No AC for timezone handling — customer and caregiver could be in different timezones (relevant for the mobile app used while traveling).
6. No AC for what happens if the payment step fails *after* the slot appears "booked" in the UI.

**Revised AC after my review (what shipped):**

- When a customer starts checkout, the slot is **held for 10 minutes** (soft lock) so it can't be booked by someone else mid-checkout.
- The booking only moves to `confirmed` status after payment succeeds; if payment fails, the slot is released back to `available` immediately and the customer sees a clear error with a retry option.
- If two customers reach checkout for the same slot at the same time, the second customer to complete payment sees "This time is no longer available" and is returned to search with the slot removed from results.
- Caregivers must accept or decline a booking request within 24 hours; if they don't respond, the booking auto-cancels and the customer is notified and refunded (if already charged).
- All booking times are stored in UTC and displayed in the viewing user's local timezone, labeled explicitly (e.g., "2:00 PM EST").
- This directly informed [BUG-002](../bug-reports/BUG-002-double-booking-race-condition.md) — a race condition where the soft-lock didn't actually prevent double-booking under concurrent load.

---

## Why this matters

Both of the AC gaps above turned into real, high-severity bugs after release (see the linked bug reports). Reviewing requirements before dev starts is the cheapest place I can prevent a bug — every gap caught in refinement is a bug that never gets written, tested, filed, triaged, fixed, and re-tested later.
