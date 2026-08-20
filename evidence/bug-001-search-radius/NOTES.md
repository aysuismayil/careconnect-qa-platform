# Evidence Notes — BUG-001 (Search radius boundary exclusion)

Referenced by [`bug-reports/BUG-001-search-radius-boundary-excluded.md`](../../bug-reports/BUG-001-search-radius-boundary-excluded.md).

Evidence collected during the real investigation (described here rather than attached as binary files — see [Evidence README](../README.md)):

- **`results-within-10mi.png`** — browser screenshot of the search results list with the "within 10 miles" filter applied, showing the boundary caregiver (seeded at exactly 10.00 mi) absent from the list. Captured via Chrome, full window, with DevTools closed for a clean UI shot.
- **`results-within-15mi.png`** — same search, filter widened to 15 miles, showing the same caregiver now present, to visually prove the caregiver record itself was fine and only the 10-mile filter excluded it.
- **`network-search-request.json`** — captured from DevTools → Network tab, "Copy → Copy response," showing the raw `/search` API response for the 10-mile query missing the boundary caregiver's ID entirely (not just hidden client-side) — this was the key piece of evidence proving the bug was server-side, not a rendering issue.
