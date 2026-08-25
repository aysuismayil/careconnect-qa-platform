BUG-001 — Caregiver at exactly 10 miles is missing from search results

**Title/Summary:** [SEARCH] Caregiver at exactly 10 miles is missing from search results

**Short Description:** When a customer filters search results by distance, a caregiver sitting exactly on the boundary (e.g. exactly 10.0 miles for a "within 10 miles" filter) does not show up. The requirement says the boundary value should be included. Right now it's dropped from the results.

**Preconditions:**
- Staging environment with seeded geolocation test data.
- A caregiver profile seeded at a distance calculated to be exactly 10.00 miles from a fixed test search origin.
- Caregiver profile is active and offers a service type included in the search.

**Steps to Reproduce:**
1. Log in as a customer test account on staging.
2. Search using the fixed test origin address: 123 Test Ave, Newark, NJ 07102 (synthetic seed address).
3. Apply the distance filter: "Within 10 miles."
4. Don't apply any other filters.
5. Check the results list for caregiver CG-1042, who is seeded at exactly 10.00 miles from the origin.

**Actual Result:** CG-1042 (10.00 mi away) does not show up in the "within 10 miles" results. It only shows up once the filter is widened to "within 15 miles."

**Expected Result:** Per the reviewed acceptance criteria (see docs/02-requirements-and-acceptance-criteria-review.md, Example 1), the radius filter should include the boundary value. A caregiver at exactly 10.00 mi should show up for a "within 10 miles" filter.

**Test Data:**
- Search origin (synthetic): 123 Test Ave, Newark, NJ 07102 → lat/long 40.7357, -74.1724 (fictional/rounded for this example)
- Seeded caregiver CG-1042, distance from origin: exactly 10.00 mi (verified independently using the haversine formula against the seeded coordinates)
- Filter applied: radius = 10 miles

**Environment/Configuration:**
- Environment: Staging
- Browser: Chrome 126 (also reproduced on Safari 17)

**Evidence/Attachments:**
- evidence/bug-001-search-radius/results-within-10mi.png — results list at 10-mile filter, CG-1042 absent
- evidence/bug-001-search-radius/results-within-15mi.png — same search at 15-mile filter, CG-1042 present
- evidence/bug-001-search-radius/network-search-request.json — sanitized copy of the search API request/response

**Severity:** Medium — no crash or data loss, but customers lose visibility of valid caregivers at common round-number radii.

**Priority:** P1 — fix before this release ships. It's a direct contradiction of signed-off acceptance criteria and affects a common search pattern.

**Repro Rate:** 5/5 — 100%. Reproduced again independently with a second caregiver seeded at exactly 5.00 mi against the "within 5 miles" filter.

**Workaround:** Customers can widen the radius filter by one step (e.g. 15 miles instead of 10) to see boundary caregivers, but that also pulls in caregivers outside the range they actually wanted. Not a clean workaround.

---

**Investigation Notes:**
- Checked the same search request directly in Postman and got the same result as the UI — the caregiver is missing from the API response itself, not just from the screen. So this isn't a frontend display issue.
- Checked DevTools → Network tab: the frontend does send radiusMiles=10 correctly.
- Caregivers at 9.99 mi and 9.5 mi both show up correctly, which points to the issue being specific to the exact boundary value, not the distance calculation in general.
- Hypothesis (not confirmed): the radius filter query may be using a "less than" comparison instead of "less than or equal to." This is a guess based on the behavior observed, not something I've verified in the code — flagging it for the dev to check.
- This scenario was called out during requirements review as an explicit acceptance criterion (see docs/02-requirements-and-acceptance-criteria-review.md).
