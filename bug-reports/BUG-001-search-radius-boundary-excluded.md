# BUG-001 — Caregivers exactly at the search radius boundary are excluded from results

**Title/Summary:**
[Search/Filtering] Caregiver located exactly at the selected radius boundary (e.g., exactly 10.0 mi for a "within 10 miles" filter) is excluded from search results, contradicting the agreed acceptance criteria.

**Short Description:**
When a customer filters caregiver search results by distance, the acceptance criteria state the radius filter should be **inclusive** of the boundary value (a caregiver exactly at 10.0 miles should appear for a "within 10 miles" filter). In testing, caregivers at exactly the boundary distance are silently dropped from results, which means customers lose visibility of legitimately eligible caregivers near common round-number radii (5, 10, 15, 25 miles).

**Preconditions:**
- Staging environment with seeded geolocation test data.
- A caregiver profile seeded at a distance calculated to be exactly 10.00 miles (straight-line/haversine) from a fixed test search origin.
- Caregiver profile is active, live, and offers a service type included in the search.

**Steps to Reproduce:**
1. Log in as a customer test account on staging.
2. Search using the fixed test origin address: `123 Test Ave, Newark, NJ 07102` (synthetic seed address).
3. Apply the distance filter: "Within 10 miles."
4. Confirm no other filters are applied.
5. Review the returned results list.
6. Cross-check against the seeded caregiver known to be at exactly 10.00 mi from the origin (`caregiver_id: CG-1042`, seeded lat/long).

**Actual Result:**
`CG-1042` (distance = 10.00 mi) does not appear in the results for the "within 10 miles" filter. It only appears once the filter is widened to "within 15 miles" or greater.

**Expected Result:**
Per the reviewed acceptance criteria (see [`docs/02-requirements-and-acceptance-criteria-review.md`](../docs/02-requirements-and-acceptance-criteria-review.md), Example 1), the radius filter should be inclusive of the boundary: a caregiver at exactly 10.00 mi should be included when the filter is "within 10 miles."

**Test Data:**
- Search origin (synthetic): `123 Test Ave, Newark, NJ 07102` → lat/long `40.7357, -74.1724` (fictional/rounded for this example)
- Seeded caregiver `CG-1042`, distance from origin: exactly `10.00` mi (computed and verified independently via haversine formula against seeded coordinates)
- Filter applied: `radius = 10` (miles)

**Environment/Configuration:**
- Environment: Staging
- Build: `release-candidate build, search service v2.3.0` (sample version label)
- Browser: Chrome 126 (also reproduced on Safari 17)
- Feature flag: `search_radius_v2` = ON

**Evidence/Attachments:**
- `evidence/bug-001-search-radius/results-within-10mi.png` — screenshot of results list at 10-mile filter, `CG-1042` absent (sanitized/synthetic screenshot, recreated for this repo, no real user data)
- `evidence/bug-001-search-radius/results-within-15mi.png` — same search at 15-mile filter, `CG-1042` present
- `evidence/bug-001-search-radius/network-search-request.json` — sanitized copy of the search API request/response payload (see [Evidence README](../evidence/README.md) for how these are sanitized)

**Severity:** Medium — no crash or data loss, but customers lose visibility of valid, bookable caregivers at common round-number radii, directly reducing potential bookings for those caregivers.

**Priority:** P1 — fix before this release ships; it's a direct contradiction of signed-off acceptance criteria and affects a common search pattern (round-number radius selection).

**Repro Rate:** 5/5 — 100% reproducible with the seeded 10.00 mi test caregiver, and reproduced again independently with a second caregiver seeded at exactly 5.00 mi against the "within 5 miles" filter.

**Workaround:** Customers can widen the radius filter by one increment (e.g., select 15 miles instead of 10) to see boundary caregivers, but this also returns caregivers they didn't intend to include — not a clean workaround.

**Investigation Notes:**
- Compared the search API response directly in Postman against the same request made through the UI — same result, so this is not a frontend display bug; the caregiver is missing from the API response itself, confirming the bug is server-side in the search/filter query.
- Reviewed the search request in DevTools → Network tab: the `radiusMiles=10` parameter is sent correctly from the frontend.
- Hypothesis, to hand off to backend dev: the distance filter query likely uses a strict `distance < radiusMiles` comparison instead of `distance <= radiusMiles`. Requesting the dev check the WHERE clause / query builder for the search endpoint's radius condition.
- Verified this is specific to the exact boundary value — caregivers at 9.99 mi and 9.5 mi both appear correctly, isolating the issue to the boundary condition itself rather than a broader distance-calculation bug.
- Linked requirement: this AC was explicitly called out during refinement (see [Requirements Review](../docs/02-requirements-and-acceptance-criteria-review.md#example-1--caregiver-search-radius-filter)) — flagging that this may have been missed during implementation because the edge case wasn't covered by unit tests either (confirmed with dev there was no boundary-specific test in the PR).
