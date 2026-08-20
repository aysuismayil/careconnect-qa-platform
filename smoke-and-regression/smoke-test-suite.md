# Smoke Test Suite

Run immediately after every deploy to staging and again on production immediately after release, before broader regression testing. Goal: fast (~15–20 min) confirmation that critical paths are not broken. If any item here fails, I stop and escalate immediately rather than continuing with a broader test pass — there's no point regression-testing a build where login is broken.

**Target runtime:** 15–20 minutes
**When run:** Every deploy to staging; immediately post-deploy to production
**Owner:** QA (me)

| # | Check | Steps | Expected | Platform |
|---|---|---|---|---|
| 1 | App/site loads | Navigate to the app URL / open the app | Loads without error, no blank screen or 500 page | Web/Mobile |
| 2 | Customer sign up works | Create a new customer account with valid data | Account created, redirected appropriately | Web |
| 3 | Customer login works | Log in with a known-good staging test account | Logged in successfully, lands on home/dashboard | Web/Mobile |
| 4 | Caregiver login works | Log in with a known-good caregiver test account | Logged in successfully, lands on caregiver dashboard | Web/Mobile |
| 5 | Search returns results | Search a known location with seeded caregiver data | Non-empty result list returned | Web/Mobile |
| 6 | Filters apply | Apply one filter (e.g., service type) | Result list narrows accordingly, no error | Web/Mobile |
| 7 | Caregiver profile loads | Open any caregiver's profile from search results | Profile loads with photo, bio, rate, availability | Web/Mobile |
| 8 | Booking checkout reachable | Select an open slot and reach the payment screen | Checkout screen loads with correct price displayed | Web/Mobile |
| 9 | Payment with test card succeeds | Complete payment using the standard test card | Booking confirms, receipt/confirmation shown | Web/Mobile |
| 10 | Booking appears in "My Bookings" | Check booking history after confirming | New booking appears with correct details | Web/Mobile |
| 11 | Cancellation works | Cancel the booking just created | Booking status updates to cancelled, refund initiated | Web/Mobile |
| 12 | Logout works | Log out | Session ends, redirected to login/landing | Web/Mobile |
| 13 | Native app launches and reaches login | Cold-launch the iOS and Android app | App launches without crash, reaches login screen | iOS/Android |
| 14 | Key third-party integrations reachable | Trigger a geolocation lookup and a test-mode payment call | Both respond successfully (not down/misconfigured) | Web/Mobile |

## Smoke test result log (sample format)

| Date | Build | Result | Notes |
|---|---|---|---|
| Sample entry | `release-1.42.0` | ✅ Pass | All 14 checks passed, cleared for regression pass |
| Sample entry | `release-1.41.3` | ❌ Fail at #9 | Payment confirmation returned 500 on staging; escalated immediately, blocked further testing until hotfixed |
