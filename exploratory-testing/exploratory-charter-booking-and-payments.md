# Exploratory Testing Charter — Booking & Payment Flow

I use session-based exploratory testing (charter + time-box + notes) to complement scripted test cases, especially in areas where scripted cases can't anticipate every real-world sequence of actions. This is the actual format I use.

---

## Charter

**Explore:** the end-to-end booking and payment flow (search → profile → book → pay → confirmation), including interruptions, back-navigation, and rapid/unexpected user actions
**With:** a customer test account, a caregiver test account I can control in a second session, and Chrome DevTools open throughout
**To discover:** issues around state consistency, race conditions, and error handling that structured test cases might not anticipate

**Time-box:** 90 minutes
**Tester:** QA (me)
**Environment:** Staging
**Session date:** sample session, recreated for this repo

---

## Session notes (test → observation format)

| # | Test idea (what I tried) | Observation | Bug filed? |
|---|---|---|---|
| 1 | Start checkout, then use browser back button repeatedly, then forward again | App state stayed consistent; slot hold was still respected. No issue. | No |
| 2 | Start checkout on two different browser tabs for the *same* customer account, same slot | Second tab showed a "you already have this slot in progress" message rather than a duplicate hold — good defensive handling | No |
| 3 | Start checkout, then change the caregiver's rate in a second session as the caregiver, then submit payment on the original screen | Confirmed the price shown did not match what was charged | **Yes — [BUG-005](../bug-reports/BUG-005-price-mismatch-frontend-backend.md)** |
| 4 | Add a caregiver to "favorites," then have that caregiver deactivate their account entirely, then revisit favorites | Favorited (deactivated) caregiver still appeared in the favorites list with a "Book Now" button that led to a broken/blank checkout page instead of a clear "no longer available" message | Logged as a lower-severity follow-up item (not detailed in this repo's 6 featured bugs, but tracked in my working notes) |
| 5 | Rapidly toggle multiple search filters on/off while results are still loading from the previous filter change | Occasionally saw a flash of stale results before the correct filtered set loaded — a minor race in how loading states are handled, not incorrect final data | Logged as low-priority UI polish item |
| 6 | Attempt to book a slot, then immediately (before confirming) navigate to a completely different caregiver's profile and start a second checkout | First checkout's hold correctly released when I abandoned it for the second; no leaked/orphaned holds observed in the DB spot-check | No |
| 7 | On mobile, start checkout, then background the app, put the phone to sleep, and return 12 minutes later (past the 10-min hold window) | Correctly showed "session expired, please select a new time" rather than a stuck/broken state — matches [TC-MOB-007](../test-cases/06-mobile-test-cases.md) expected behavior | No |
| 8 | Double-click the "Pay Now" button as fast as possible, then also try rapid-tapping on mobile | Same underlying issue as duplicate-charge investigation — see dedicated bug reports for the two distinct root causes found | **Yes — [BUG-003](../bug-reports/BUG-003-payment-double-charge-retry.md)**, **[BUG-006](../bug-reports/BUG-006-mobile-duplicate-charge-on-reconnect.md)** |
| 9 | Enter a booking note/message field with emoji, right-to-left text, and a very long string (2,000+ chars) | Handled correctly — emoji and RTL text displayed properly; long text was truncated with a visible counter, no crash or data corruption | No |
| 10 | Cancel a booking, then immediately try to re-book the exact same slot before the cancellation fully processed | Brief window (~1–2 sec) where the slot showed as unavailable even though cancellation should have freed it — resolved itself without a refresh; noted as a minor UX delay, not a functional bug (self-corrected within the same session) | Logged as low-priority observation, not filed as a full bug |

---

## Session summary

- 2 new bugs directly discovered through exploratory testing that scripted test cases had not caught: BUG-005 (price mismatch) and reinforced discovery of the double-charge pattern that led to BUG-003/BUG-006.
- Several lower-priority UX/polish observations logged separately for the team's backlog.
- Confirmed several "should be fine" assumptions really do hold (hold/lock behavior across tabs, mobile backgrounding behavior) — exploratory testing is as valuable for confirming resilience as it is for finding new bugs.
- This session's most valuable technique: deliberately introducing a **second actor** (a caregiver session acting concurrently with a customer session) rather than testing each role in isolation — several of the most serious bugs on this project (BUG-002, BUG-005) only appear when two users interact with the same resource at nearly the same time, which single-actor scripted test cases naturally don't cover.

## How I decide what to explore

I bias exploratory time toward:
1. **Areas with real money/business risk** (payments, booking confirmation) over cosmetic areas.
2. **Multi-actor scenarios** (two customers, or a customer + caregiver interacting with the same resource concurrently) since these are the hardest to anticipate in single-user scripted cases.
3. **Interruption and recovery** (network drops, backgrounding, back-button, session expiry) since real users don't follow the "happy path" a test case assumes.
4. **Areas that just changed** — new code is where new bugs live.
