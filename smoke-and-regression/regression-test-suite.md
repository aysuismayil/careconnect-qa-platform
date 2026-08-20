# Regression Test Suite

Run before each release (full pass) and in a targeted/scoped form after any bug fix (only the affected area(s) plus adjacent flows that share code paths). Pulls from the detailed cases in [`test-cases/`](../test-cases/) — this document is the **checklist/tracking view** used to plan and record a regression pass, not a duplicate of the detailed steps.

## Full regression scope (pre-release)

| Area | Test cases covered | Est. time |
|---|---|---|
| Authentication | All of [`01-authentication-test-cases.md`](../test-cases/01-authentication-test-cases.md) | 45 min |
| Search & Filtering | All of [`02-search-filtering-test-cases.md`](../test-cases/02-search-filtering-test-cases.md) | 40 min |
| Caregiver Profiles | All of [`03-caregiver-profile-test-cases.md`](../test-cases/03-caregiver-profile-test-cases.md) | 35 min |
| Booking | All of [`04-booking-test-cases.md`](../test-cases/04-booking-test-cases.md) | 50 min |
| Payments | All of [`05-payments-test-cases.md`](../test-cases/05-payments-test-cases.md) | 40 min |
| Mobile | All of [`06-mobile-test-cases.md`](../test-cases/06-mobile-test-cases.md) | 60 min |
| API regression | Full [Postman collection](../api-testing/postman/CareConnect-API.postman_collection.json) via Collection Runner | 15 min |
| Accessibility spot-check | New/changed screens only, from [`accessibility-test-cases.md`](../accessibility-testing/accessibility-test-cases.md) | 20 min |

**Total estimated full regression time:** ~5 hours manual + 15 min automated-via-Runner. On a tight release timeline, I prioritize P0/P1 cases first (see [risk-based prioritization](../docs/01-test-plan-and-strategy.md#8-risk-based-prioritization)) and treat P2/P3 as time-permitting.

## Targeted regression (after a bug fix — example)

When a fix ships for a specific bug, I don't just re-test that one case — I test the fix plus everything that shares a code path, since fixes are a common source of new regressions.

**Example: after the fix for [BUG-002](../bug-reports/BUG-002-double-booking-race-condition.md) (double-booking race condition):**

| Re-test | Why |
|---|---|
| TC-BOOK-004 (the original failing case) | Confirm the specific bug is actually fixed |
| TC-BOOK-002, TC-BOOK-003 | Same soft-lock mechanism — confirm the fix didn't break normal hold/release behavior |
| TC-BOOK-005 | Confirm slot release-on-payment-failure still works after the concurrency fix |
| TC-BOOK-009, TC-BOOK-011 | Cancellation and rescheduling also touch slot state — confirm no side effects |
| API-BOOK-02 (Postman) | Re-run the concurrent-confirm repro via Collection Runner to confirm it no longer succeeds twice |
| SQL Query 1 (overlapping bookings) | Confirm zero overlapping confirmed bookings after a batch of concurrent test attempts |

## Regression pass result log (sample format)

| Date | Scope | Pass/Fail | Notes |
|---|---|---|---|
| Sample entry | Full pre-release regression, `release-1.42.0` | 96% pass (2 P2 issues found, no P0/P1) | Both P2 issues accepted as known-risk by PM for this release; see [Release Readiness](../docs/03-release-readiness-and-test-summary.md) |
| Sample entry | Targeted — BUG-002 fix verification | Pass | All listed re-test cases passed; SQL check confirmed zero overlapping bookings across 20 concurrent test attempts |

## What I automate vs. keep manual

I flag strong candidates for the SDET's automated suite when a case is: stable (not changing frequently), high-value to run on every build, and expensive to repeat manually every release (e.g., the full authentication suite, core search filter combinations). I keep manual: anything involving visual/UX judgment, exploratory-style edge cases, accessibility checks requiring a human with a screen reader, and newly-changed areas where automation coverage doesn't exist yet.
